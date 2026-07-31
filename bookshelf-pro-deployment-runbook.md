# BookShelf Pro — Jenkins → ECR → ArgoCD → EKS Deployment Runbook

Full from-scratch steps for deploying the BookShelf Pro app using the pipeline:
**Developer → GitHub → Jenkins → Docker build → ECR → sed-update K8s manifest → Git push → ArgoCD sync → EKS pods → LoadBalancer / Route 53**

Repo used: `https://github.com/Damodar-paligili/2nd10WeeksofCloudOps-main.git`
AWS account: `815512685331`, region `us-east-1`

---

## Prerequisites

- An EKS cluster already created (`kubectl` working against it)
- One EC2 instance with an IAM instance role that has EKS/ECR/Route53 permissions (this doubles as the Jenkins server)
- A forked copy of the repo under your own GitHub account

---

## Phase 1 — Install Docker, Java, Jenkins on the EC2 instance

```bash
dnf update -y
dnf install docker -y
systemctl enable --now docker

# Jenkins requires Java 21 — install both 17 (JRE) and 21 devel (JDK, for the Jenkins 'jdk' tool)
dnf install -y java-17-amazon-corretto
dnf install -y java-21-amazon-corretto-devel

wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.repo/jenkins.io-2023.key
dnf install -y jenkins
usermod -aG docker jenkins

# Point Jenkins at Java 21 (the base package only installs a JRE Jenkins can't use)
mkdir -p /etc/systemd/system/jenkins.service.d
cat > /etc/systemd/system/jenkins.service.d/override.conf << 'EOF'
[Service]
Environment="JAVA_HOME=/usr/lib/jvm/java-21-amazon-corretto.x86_64"
Environment="PATH=/usr/lib/jvm/java-21-amazon-corretto.x86_64/bin:/usr/bin:/bin:/usr/sbin:/sbin"
EOF

systemctl daemon-reload
systemctl enable --now jenkins
systemctl restart docker
systemctl status jenkins --no-pager
```

Open port **8080** in the EC2 security group (inbound TCP, your IP or 0.0.0.0/0).

Get the initial admin password and log in at `http://<ec2-public-ip>:8080`:
```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```
Choose **"Install suggested plugins"** and create your admin user.

---

## Phase 2 — Jenkins plugins and tools

**Manage Jenkins → Plugins → Available plugins** → install:
- **NodeJS** plugin

**Manage Jenkins → Tools**:
- **JDK installations** → Add JDK → Name: `jdk` → uncheck "Install automatically" → JAVA_HOME: `/usr/lib/jvm/java-21-amazon-corretto.x86_64`
- **NodeJS installations** → Add NodeJS → Name: `nodejs` → check "Install automatically" → Version: latest 20.x

Confirm IAM role auth works (no static keys needed if the instance has a role):
```bash
aws sts get-caller-identity
```

---

## Phase 3 — Create ECR repositories

```bash
aws ecr create-repository --repository-name backend --region us-east-1
aws ecr create-repository --repository-name frontend --region us-east-1
aws ecr describe-repositories --region us-east-1
```

---

## Phase 4 — Fix the forked repo's files

Edit these directly on GitHub (pencil icon → edit → commit to `main`), replacing the org placeholder (`CloudTechDevOps`) with your own username, and fixing the bugs below.

### `Jenkins-Pipeline-Code/Jenkinsfile-Backend`
- Comment out `SCANNER_HOME=tool 'sonar-scanner'`, the `Sonarqube Analysis`, `Quality Check`, `Trivy File Scan`, and `TRIVY Image Scan` stages
- Change git URL / `GIT_USER_NAME` to your fork
- Remove the duplicate mid-pipeline `Checkout Code` stage

### `Jenkins-Pipeline-Code/Jenkinsfile-Frontend`
- Same Sonar/Trivy stages commented out, git URL/user updated
- Fix the `sed` target: it originally pointed at a nonexistent `frontend-deploy-service.yaml` — must be `frontend-deployment.yml`

### `client/nginx.conf`
- Bug: `proxy_pass http://backendip:port;` (literal placeholder, never replaced)
- Fix: `proxy_pass http://backend:80;` (matches the K8s Service name `backend`)

### `kubernetes-files/backend-deployment.yaml`
- `image:` was a leftover GCP Artifact Registry path — set to `815512685331.dkr.ecr.us-east-1.amazonaws.com/backend:latest` (Jenkins overwrites the tag on first successful run)
- `replicas: 4` → `2` (right-sized for a small cluster)

### `kubernetes-files/frontend-deployment.yml`
- Same `image:` fix → `815512685331.dkr.ecr.us-east-1.amazonaws.com/frontend:latest`
- Removed unused commented-out `REACT_APP_API_BASE_URL` env block
- `type: LoadBalancer` kept as-is (this is how the app is reached in a browser)

`secret.yaml` was already correct — decodes to `db_host: mysql-service`, `db_user: root`, `db_password: root`, consistent with `mysql-deployment.yaml`'s `MYSQL_ROOT_PASSWORD: root`. No change needed.

---

## Phase 5 — Jenkins credentials

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials**, kind = **Secret text** for each:

| ID (exact) | Secret |
|---|---|
| `ACCOUNT_ID` | `815512685331` |
| `ECR_REPO1` | `frontend` |
| `ECR_REPO2` | `backend` |
| `my-git-pattoken` | GitHub PAT (classic, `repo` scope) |

Generate the PAT: GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token (classic) → check `repo` → Generate → copy immediately.

---

## Phase 6 — Create Jenkins pipeline jobs

For each of `bookshelf-backend` and `bookshelf-frontend`:

1. **New Item** → name it → select **Pipeline** → OK
2. **Pipeline** section:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: `https://github.com/<you>/2nd10WeeksofCloudOps-main.git`
   - Credentials: `- none -` (public repo)
   - Branch Specifier: `*/main`
   - Script Path: `Jenkins-Pipeline-Code/Jenkinsfile-Backend` (or `-Frontend`)
3. Save → **Build Now**

Verify after each run:
```bash
aws ecr describe-images --repository-name backend --region us-east-1
aws ecr describe-images --repository-name frontend --region us-east-1
```
And check the `image:` line in `backend-deployment.yaml` / `frontend-deployment.yml` on GitHub — it should now show the ECR path with a build-number tag (e.g. `:1`).

---

## Phase 7 — Install ArgoCD on EKS

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w   # wait for all Running, Ctrl+C when done

# Expose the UI
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd   # note the EXTERNAL-IP

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Log into `http://<argocd-elb-hostname>` as `admin` / that password.

---

## Phase 8 — Create the ArgoCD Application

**+ NEW APP**:
- Application Name: `bookshelf-pro`
- Project: `default`
- Sync Policy: `Automatic`, check **Prune Resources** and **Self Heal**
- Repository URL: `https://github.com/<you>/2nd10WeeksofCloudOps-main.git`
- Revision: `main`
- Path: `kubernetes-files`
- Cluster URL: `https://kubernetes.default.svc`
- Namespace: `default`
- **CREATE**

Verify everything reaches **Synced** / **Healthy**, then confirm pods:
```bash
kubectl get pods
kubectl get svc frontend   # note the EXTERNAL-IP (ELB hostname)
```

Open `http://<frontend-elb-hostname>` — the BookShelf Pro app should load with the seeded sample books.

---

## Phase 9 — Custom domain via Route 53 (optional)

```bash
aws route53 create-hosted-zone --name <yourdomain> --caller-reference $(date +%s)
```
Note the 4 `NameServers` returned.

In your domain registrar (e.g. GoDaddy): **DNS → Nameservers → Change Nameservers → Enter my own (advanced)** → paste in those 4 nameservers → Save.

In Route 53, in the new hosted zone → **Create record**:
- Record name: blank (root) or a subdomain like `app`
- Type: `A`
- Alias: ON → Alias to Application and Classic Load Balancer → `us-east-1` → select the frontend's ELB

Wait for propagation (15 min–a few hours), then verify:
```bash
dig <yourdomain-or-subdomain>
```
Look for a populated **ANSWER SECTION**. Once resolved, `http://<yourdomain>` loads the app.

---

## Day-2 workflow (after setup)

1. Push code changes to `backend/` or `client/` in your fork
2. Trigger (or wait for a webhook/poll to trigger) the matching Jenkins job
3. Jenkins builds the image, pushes to ECR, updates the manifest's `image:` tag, commits back to `kubernetes-files/`
4. ArgoCD detects the Git change and auto-syncs, rolling out the new pod
5. Refresh the browser — the change is live
