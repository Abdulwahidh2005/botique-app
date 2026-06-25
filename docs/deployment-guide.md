# Boutique App — End-to-End Deployment Guide

A step-by-step, **study-friendly** walkthrough of how the Boutique microservices app is deployed to AWS EKS using **Terraform** (infrastructure) + **GitHub Actions** (CI) + **Argo CD** (GitOps delivery).

Every command is followed by a plain-language explanation of **what it does** and **why** you run it. The last section documents the real errors hit during a first deployment and how each was fixed — keep it as a troubleshooting reference.

---

## 0. The big picture

```
        ┌─────────────┐   terraform apply   ┌──────────────────────────────┐
        │  Terraform  │ ──────────────────► │ AWS: VPC, EKS, Node Group,    │
        │  (IaC)      │                     │ ECR repos, IAM roles          │
        └─────────────┘                     └──────────────────────────────┘
                                                          │
        ┌─────────────┐  build + push image   ┌───────────▼───────┐
        │ GitHub      │ ────────────────────► │ Amazon ECR        │
        │ Actions(CI) │  update manifest tag  │ (container images)│
        └─────┬───────┘                       └───────────────────┘
              │ commit new image tag to git
              ▼
        ┌─────────────┐   watches git repo    ┌───────────────────┐
        │   Git repo  │ ◄──────────────────── │     Argo CD       │
        │ (gitops/)   │       syncs to ───────►│   (in cluster)   │
        └─────────────┘                       └─────────┬─────────┘
                                                        │ kubectl apply
                                                        ▼
                                              ┌───────────────────┐
                                              │  EKS cluster:     │
                                              │  frontend, gateway│
                                              │  auth, orders,    │
                                              │  postgres, etc.   │
                                              └───────────────────┘
```

**Mental model:**
- **Terraform** builds the *empty house* (network + Kubernetes cluster + image registry).
- **GitHub Actions (CI)** *manufactures the furniture* (builds Docker images, pushes them to ECR) and writes down *which furniture goes where* (updates image tags in the Kubernetes manifests).
- **Argo CD** is the *mover* that watches the git repo and puts the furniture in the house (deploys manifests to the cluster).

---

## 1. Prerequisites (install once)

| Tool | What it's for |
|------|---------------|
| AWS CLI | Talk to AWS from the terminal |
| Terraform (>= 1.5) | Create the AWS infrastructure |
| kubectl | Talk to the Kubernetes cluster |
| helm | (Used by Terraform) install Argo CD + monitoring |
| git | Source control / GitOps |

Verify they exist:

```bash
aws --version
terraform version
kubectl version --client
helm version
```
**What this does:** prints each tool's version. **Why:** confirms they're installed and on your `PATH` before you start — cheaper to find out now than mid-deploy.

---

## 2. Configure AWS credentials (use an IAM user, NOT root)

> ⚠️ **Never run this with your AWS account *root* user.** EKS cannot map the root user to a Kubernetes admin, so you'll get `Unauthorized` errors later. Always use a normal IAM user.

### 2a. Create an IAM user (once)
```bash
aws iam create-user --user-name terraform-admin
aws iam attach-user-policy \
  --user-name terraform-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam create-access-key --user-name terraform-admin
```
**What this does:**
1. Creates an IAM user named `terraform-admin`.
2. Grants it full admin permissions.
3. Generates an **access key + secret key** (printed once — save them).

**Why:** Terraform and `kubectl` need an identity AWS recognizes. A dedicated IAM user is safer than root and is the identity EKS will treat as cluster admin.

### 2b. Point your CLI at that user
```bash
aws configure
# AWS Access Key ID:     <paste>
# AWS Secret Access Key: <paste>
# Default region name:   us-east-1
```
**What this does:** stores the keys in `~/.aws/credentials` so every `aws`/`terraform` command authenticates as this user.

### 2c. Confirm who you are
```bash
aws sts get-caller-identity
```
**What this does:** prints the IAM identity your credentials resolve to. **Why:** make sure the `Arn` says `user/terraform-admin` and **not** `:root`.

---

## 3. Provision the infrastructure with Terraform

All commands run from the `projects/Infrastructure` folder.

```bash
cd projects/Infrastructure
```

### 3a. Initialize
```bash
terraform init
```
**What this does:** downloads the provider plugins (AWS, Kubernetes, Helm, TLS) and sets up the working directory. **Why:** Terraform can't plan or apply until its providers are installed.
> 💡 If a provider download times out (`context deadline exceeded`), it's a slow-network issue — just run `terraform init` again; it resumes where it left off.

### 3b. Preview the changes
```bash
terraform plan
```
**What this does:** shows exactly what Terraform *would* create/change/destroy, without doing it. **Why:** a safety check — read it before applying.

### 3c. Create everything
```bash
terraform apply
# type: yes
```
**What this does:** builds the VPC, subnets, the EKS cluster, the worker node group, the ECR repositories, and all IAM roles. It also installs Argo CD and the kube-prometheus-stack via Helm.
**Why:** this is the actual infrastructure creation step. Takes ~15–20 min (the monitoring stack alone is large).

Key variables live in `terraform.tfvars` (region, cluster name, node size/count, ECR repo list).

---

## 4. Grant your IAM user access to the cluster

A freshly created EKS cluster only trusts its creator. If you ever see `Unauthorized` from `kubectl`, run these.

### 4a. Switch the cluster to support "access entries"
```bash
aws eks update-cluster-config \
  --name eks-cluster \
  --access-config authenticationMode=API_AND_CONFIG_MAP
```
**What this does:** enables the modern **access entries** auth method (managed via the AWS API). **Why:** the old `CONFIG_MAP` method requires editing an in-cluster file you can't reach yet — a chicken-and-egg trap. This is a one-way, non-destructive change.

### 4b. Grant your user cluster-admin
```bash
aws eks create-access-entry \
  --cluster-name eks-cluster \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:user/terraform-admin

aws eks associate-access-policy \
  --cluster-name eks-cluster \
  --principal-arn arn:aws:iam::<ACCOUNT_ID>:user/terraform-admin \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```
**What this does:** registers your IAM user on the cluster, then attaches the cluster-admin policy. **Why:** this is what makes `kubectl` commands authorized.

> 📌 These steps are currently **manual** (done with the CLI, not in Terraform). That means a fresh `terraform apply` won't recreate them — see the "Known gotchas" section.

---

## 5. Connect kubectl to the cluster

```bash
aws eks update-kubeconfig --name eks-cluster --region us-east-1
```
**What this does:** writes the cluster's connection details into `~/.kube/config` and switches your current context to it. **Why:** `kubectl` now knows *which* cluster to talk to and how to authenticate.

Verify:
```bash
kubectl get nodes
```
**What this does:** lists the worker nodes. **Why:** if you see nodes in `Ready` state, your access works.

---

## 6. Build & push images with GitHub Actions (CI)

The cluster is empty of *your* images until CI builds and pushes them.

### 6a. Add repository secrets on GitHub
Go to **GitHub repo → Settings → Secrets and variables → Actions** and add:

| Secret | Example value | Used for |
|--------|---------------|----------|
| `AWS_ACCESS_KEY_ID` | `AKIA...` | Auth to AWS / push to ECR |
| `AWS_SECRET_ACCESS_KEY` | `...` | (the matching secret) |
| `AWS_REGION` | `us-east-1` | Which region's ECR |
| `AWS_ACCOUNT_ID` | `979570201162` | Builds the ECR image URL |

> `GITHUB_TOKEN` is **auto-provided** by GitHub — do not create it.

### 6b. Run the pipeline
**Actions tab → "Boutique CI Pipeline" → Run workflow.**

**What the pipeline does (see `.github/workflows/ci.yml`):**
1. **build-and-push job** — for each service: logs in to ECR, builds the Docker image tagged with the git commit SHA, and pushes it to ECR.
2. **update-manifests job** — rewrites the `image:` tag in each `gitops/k8s/**` manifest to the new SHA, then commits and pushes that change back to the repo.

**Why:** this keeps git as the single source of truth — the manifests always describe exactly which image version should be running.

---

## 7. Deploy with Argo CD (GitOps)

Argo CD watches the `gitops/` folder and reconciles the cluster to match it.

### 7a. Open the Argo CD UI
```bash
kubectl port-forward svc/argocd-server -n argocd 9090:80
```
**What this does:** tunnels your local port `9090` to the Argo CD service inside the cluster. **Why:** the service is `ClusterIP` (internal only), so port-forward is the simplest way to reach it.

Then open **`http://localhost:9090`** in your browser.
> ⚠️ Use **`http`**, not `https`. This deployment runs Argo CD in *insecure* (plain-HTTP) mode (`server.insecure=true`). A `https://` request to a HTTP server gives `connection reset by peer`.

### 7b. Get the admin password
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```
**What this does:** reads the auto-generated initial admin password from a Kubernetes Secret and decodes it from base64. **Why:** you log in as `admin` with this password.

### 7c. Sync the app
In the UI, open the `boutique` app and click **Sync** (or via CLI):
```bash
kubectl -n argocd patch application boutique \
  --type merge -p '{"operation":{"sync":{"revision":"HEAD"}}}'
```
**What this does:** tells Argo CD to apply the current git manifests to the cluster now. **Why:** this app has **no auto-sync**, so deployments are manual until you enable it.

---

## 8. Bootstrap the databases

The Postgres pod starts with only a default `postgres` database, but each service needs its own. Create them once:

```bash
for db in auth_db products_db orders_db users_db; do
  kubectl exec -n boutique boutique-postgres-0 -- \
    psql -U postgres -c "CREATE DATABASE $db;"
done
```
**What this does:** runs `CREATE DATABASE` inside the running Postgres pod for each service database. **Why:** without these, services like `order-service` crash with `database "orders_db" does not exist`.

> 📌 This is currently **manual**. The fix to make it automatic is to mount the `postgres-init-scripts` ConfigMap into the Postgres StatefulSet — see "Known gotchas".

---

## 9. Verify everything is healthy

```bash
kubectl get pods -n boutique
```
**What this does:** lists all app pods. **Why:** you want every pod `1/1 Running`.

```bash
kubectl get application boutique -n argocd
```
**What this does:** shows Argo CD's view. **Why:** you want `Synced` + `Healthy`.

Reach the app:
```bash
kubectl port-forward svc/frontend -n boutique 8080:80
# open http://localhost:8080
```
**What this does:** tunnels to the frontend service so you can use the app in a browser.

---

## 10. Tear down (to stop paying for it)

```bash
cd projects/Infrastructure
terraform destroy
# type: yes
```
**What this does:** deletes everything Terraform created — cluster, nodes, VPC, **and the ECR repos + images** (because `force_delete = true`). **Why:** EKS + EC2 + NAT cost money hourly; destroy when you're done practicing.

---

## 11. Recreating after a destroy — important order

A fresh `destroy` + `apply` will **re-trigger two errors** unless you follow this order:

1. `terraform apply` — recreates infra and **empty** ECR repos.
2. **Run the CI pipeline** — rebuilds images into the empty ECR and updates manifest tags to a new SHA.
   *(If you skip this, Argo CD deploys image tags that no longer exist → `ImagePullBackOff`.)*
3. **Create the databases** (Section 8) — the Postgres volume was destroyed, so the DBs are gone again.
4. **Sync Argo CD.**

| Error | Recurs on destroy+reapply? | Why |
|-------|----------------------------|-----|
| `InvalidImageName` | ❌ No — real account ID is hardcoded in git | — |
| `ImagePullBackOff` | ⚠️ Yes, until CI runs | ECR emptied; images must be rebuilt + the manifest tag must match a real image |
| DB `does not exist` | ⚠️ Yes | Postgres volume destroyed; init scripts not wired into the StatefulSet |

---

## 12. Troubleshooting reference (real errors from the first deploy)

### `Error: Failed to install provider ... context deadline exceeded`
- **Cause:** slow/unstable network downloading a Terraform provider.
- **Fix:** rerun `terraform init` (it resumes). Disable VPN/proxy if it persists.

### `Error: Unauthorized` when creating Kubernetes resources
- **Cause:** Terraform/kubectl is running as the AWS **root user**, which EKS won't map to a k8s admin.
- **Fix:** use an IAM user (Section 2) and grant it an access entry (Section 4).

### Helm `failed to download openapi: context deadline exceeded`
- **Cause:** a huge chart (kube-prometheus-stack) downloading the cluster's OpenAPI schema over a slow link.
- **Fix:** set `disable_openapi_validation = true` on the `helm_release`. For the monitoring chart, also disable the operator admission webhooks (`prometheusOperator.admissionWebhooks.enabled = false`, `prometheusOperator.tls.enabled = false`), whose pre-install hooks also time out.

### Pods stuck in `InvalidImageName`
- **Cause:** the manifest `image:` contained an unsubstituted placeholder like `<AWS_ACCOUNT_ID>` — illegal characters for an image name.
- **Fix:** replace with the real account ID. The CI tag-update only changes the tag and assumes the real account ID is already in the manifest.

### Pods stuck in `Pending`
- **Cause:** the single small node hit its **max-pods limit (~29)** — each pod needs an IP from the node's ENIs (AWS VPC CNI). CPU/memory may still look free.
- **Fix:** add a node (`desired_size` 1 → 2 in `terraform.tfvars`, then `terraform apply`).

### Pod `CrashLoopBackOff` — `database "X" does not exist` (code 3D000)
- **Cause:** the service's database was never created.
- **Fix:** create the databases (Section 8), then `kubectl rollout restart deployment -n boutique <name>`.

### Argo CD port-forward — `connection reset by peer`
- **Cause:** opening `https://` against an insecure (HTTP-only) Argo CD server.
- **Fix:** use `http://localhost:9090`.

---

## 13. Known gotchas to fix for a fully reproducible deploy

These were patched live during the first deploy but aren't yet correct in code:

1. **Database init is manual.** Mount the `postgres-init-scripts` ConfigMap into the Postgres StatefulSet at `/docker-entrypoint-initdb.d/` so a fresh volume auto-creates the databases. (Init scripts only run on an empty data directory.)
2. **Cluster access setup is manual.** The auth-mode switch + access entry (Section 4) are CLI steps. Encode them in Terraform with an `access_config` block on `aws_eks_cluster` plus `aws_eks_access_entry` / `aws_eks_access_policy_association` resources.
3. **No Argo CD auto-sync.** Enable `syncPolicy.automated` so CI pushes deploy without a manual Sync.

---

## Quick command cheat-sheet

```bash
# Auth
aws configure
aws sts get-caller-identity

# Infra
cd projects/Infrastructure
terraform init
terraform plan
terraform apply

# Cluster access
aws eks update-kubeconfig --name eks-cluster --region us-east-1
kubectl get nodes

# Inspect
kubectl get pods -n boutique
kubectl get application boutique -n argocd
kubectl logs -n boutique <pod>
kubectl describe pod -n boutique <pod>

# Argo CD
kubectl port-forward svc/argocd-server -n argocd 9090:80   # then http://localhost:9090
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Databases
for db in auth_db products_db orders_db users_db; do
  kubectl exec -n boutique boutique-postgres-0 -- psql -U postgres -c "CREATE DATABASE $db;"
done

# Teardown
terraform destroy
```
