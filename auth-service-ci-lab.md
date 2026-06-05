# Lab: auth-service CI/CD + GitOps — End to End

You will wire up a complete delivery pipeline for `auth-service` — from a git push on your laptop to a running pod on Kubernetes, with ArgoCD ensuring the cluster always matches git.

```
feature branch push
      │
      ▼
GitHub Actions PR check
  (tests + SAST, ~5 min, no Docker)
      │
      ▼
merge to develop
      │
      ▼
GitHub Actions full CI
  ├─ Maven tests + CodeQL + Semgrep
  ├─ Docker build → Trivy scan
  ├─ Push to ECR  (sha-xxxxxxx)
  └─ Cosign sign
      │
      ├──► deploy-dev job ──► git commit to zen-gitops-lab1
      │                         envs/dev/values-auth-service.yaml
      │                                  │
      │                                  ▼
      │                         ArgoCD detects commit
      │                         helm template → kubectl apply
      │                         DEV pod rolling to sha-xxxxxxx ✓
      │
      └──► open-qa-pr job ──► PR opened in zen-gitops-lab1
                                envs/qa/values-auth-service.yaml
```

---

## What your instructor has already set up

| Resource | Details |
|---|---|
| EKS cluster | `pharma-dev` — you have `kubectl` access |
| ECR repository | `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/auth-service` |
| IAM role | `pharma-dev-github-actions-role` (GitHub OIDC — no static keys needed) |
| RDS PostgreSQL | `<DB_HOST>` — running in the same VPC as EKS |
| Kubernetes Secrets | `db-credentials` and `jwt-secret` pre-created in the `dev` namespace |
| ArgoCD | Installed on EKS, app `auth-service-dev` created pointing at `https://github.com/<YOUR-USERNAME>/zen-gitops-lab1.git` |

Your instructor will give you:
- `<ACCOUNT_ID>` — 12-digit AWS account number
- `<DB_HOST>` — RDS endpoint for dev
- `<QA_DB_HOST>` — RDS endpoint for qa
- kubeconfig file for `kubectl` access
- ArgoCD UI URL and login

---

## Prerequisites

Install these on your laptop before the lab:

- `git` — `git --version`
- `kubectl` — `kubectl version --client`
- `argocd` CLI — `brew install argocd` / [Windows install](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- AWS CLI — `aws --version` (for ECR verification in Part 4)

Configure your kubeconfig (instructor will provide the command):

```bash
aws eks update-kubeconfig --region us-east-1 --name pharma-dev
kubectl get nodes   # should list cluster nodes
```

---

## Part 1 — Fork both repos

Do this first. You need both repos forked before configuring anything else.

### Step 1.1 — Fork zen-pharma-backend-lab1

1. Go to `https://github.com/DPP-2026/zen-pharma-backend-lab1`
2. Click **Fork** → select your account → **Create fork**
3. Clone your fork:

```bash
git clone https://github.com/<YOUR-USERNAME>/zen-pharma-backend-lab1.git
cd zen-pharma-backend-lab1
```

Your fork contains:

```
zen-pharma-backend-lab1/
├── auth-service/
│   ├── src/
│   ├── pom.xml
│   ├── checkstyle.xml
│   └── Dockerfile
└── .github/
    └── workflows/
        ├── _java-pr-check.yml      ← reusable: tests + SAST (no Docker)
        ├── _java-build.yml         ← reusable: full build + ECR push
        ├── ci-pr-auth-service.yml  ← trigger: feature branches / PRs
        └── ci-auth-service.yml     ← trigger: develop merges
```

You do not create any workflow files — they are already there.

### Step 1.2 — Fork zen-gitops-lab1

1. Go to `https://github.com/DPP-2026/zen-gitops-lab1`
2. Click **Fork** → select your account → **Create fork**
3. Clone your fork in a separate directory:

```bash
cd ..
git clone https://github.com/<YOUR-USERNAME>/zen-gitops-lab1.git
cd zen-gitops-lab1
```

---

## Part 2 — Configure GitHub Actions

All configuration below is in GitHub Settings — no code changes.

### Step 2.1 — Create a Personal Access Token for GitOps writes

The CI pipeline commits to your `zen-gitops-lab1` repo after every build. It needs a token to do that.

1. GitHub → your profile icon → **Settings**
2. Left sidebar → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
3. Click **Generate new token**
4. Fill in:
   - **Token name**: `zen-pharma-gitops-token`
   - **Expiration**: 90 days
   - **Resource owner**: your account
   - **Repository access**: select only `zen-gitops-lab1` (your fork)
5. Under **Permissions**, set:
   - **Contents**: Read and write
   - **Pull requests**: Read and write
6. Click **Generate token** — **copy it now**, you will not see it again

### Step 2.2 — Add GitHub Secrets to zen-pharma-backend-lab1

Go to your `zen-pharma-backend-lab1` fork → **Settings** → **Secrets and variables** → **Actions** → **Secrets** tab.

Click **New repository secret** and add:

| Secret name | Value |
|---|---|
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account ID (from instructor) |
| `GITOPS_TOKEN` | The PAT you created in Step 2.1 |

### Step 2.3 — Add a GitHub Variable

Still in **Secrets and variables** → click the **Variables** tab → **New repository variable**:

| Variable name | Value |
|---|---|
| `GITOPS_REPO` | `<YOUR-USERNAME>/zen-gitops-lab1` |

---

## Part 3 — Update GitOps values files

The CI pipeline writes the new image tag into these files after every build. ArgoCD reads them to know what to deploy.

Both files already exist in your `zen-gitops-lab1` fork. You only need to fill in the two placeholder values your instructor gave you.

### Step 3.1 — Update envs/dev/values-auth-service.yaml

```bash
cd zen-gitops-lab1
```

Open `envs/dev/values-auth-service.yaml` and replace:

- `<ACCOUNT_ID>` → your 12-digit AWS account ID
- `<DB_HOST>` → the RDS dev endpoint from your instructor

### Step 3.2 — Update envs/qa/values-auth-service.yaml

Open `envs/qa/values-auth-service.yaml` and replace:

- `<ACCOUNT_ID>` → your 12-digit AWS account ID
- `<QA_DB_HOST>` → the RDS QA endpoint from your instructor

### Step 3.3 — Commit and push

```bash
git add envs/dev/values-auth-service.yaml envs/qa/values-auth-service.yaml
git commit -m "feat(auth-service): add dev and qa values files"
git push origin main
```

ArgoCD will now detect these files within ~3 minutes. Since there is no image yet (`tag: latest`), the pod may fail to start — that is expected. The next part fixes it.

---

## Part 4 — Trigger the CI pipeline

### Step 4.1 — Create the develop branch

```bash
cd zen-pharma-backend-lab1
git checkout -b develop
git push -u origin develop
```

### Step 4.2 — Push to develop and watch the full CI

```bash
git checkout develop
echo "" >> auth-service/pom.xml
git add auth-service/pom.xml
git commit -m "test: trigger full CI"
git push origin develop
```

Go to **Actions** → **CI/CD — auth-service**. You will see 3 jobs:

---

**Job 1 — Build & Security Gates** (~12–15 min)

Watch the steps scroll by:

```
✓ Checkout
✓ Set image tag  →  sha-abc1234
✓ Setup Java 17 (Temurin)
✓ Start PostgreSQL container
✓ Initialize CodeQL
✓ Maven verify  (tests pass)
✓ Upload JaCoCo coverage report
✓ CodeQL analyze
✓ Semgrep SAST
✓ Configure AWS credentials (OIDC — no static keys)
✓ Login to Amazon ECR
✓ Build Docker image
✓ Trivy image scan  (no HIGH/CRITICAL CVEs)
✓ Upload Trivy SARIF → GitHub Security tab
✓ Push image to ECR  (sha-abc1234)
✓ Install Cosign
✓ Cosign sign image  (keyless → Rekor transparency log)
```

---

**Job 2 — Deploy DEV** (starts immediately after Job 1)

Watch the logs:

```
Run yq e ".image.tag = \"sha-abc1234\"" -i "_gitops/envs/dev/values-auth-service.yaml"
[main xyz5678] ci(dev): update auth-service → sha-abc1234
```

Then open your `zen-gitops-lab1` fork on GitHub. You will see a new commit on `main`:
```
ci(dev): update auth-service → sha-abc1234
```

---

**Job 3 — Open PR QA Promotion** (starts after Job 2)

Open your `zen-gitops-lab1` fork → **Pull requests**. You will see:
```
promote(qa): auth-service → sha-abc1234
```

Leave this PR open for now — you will merge it in Part 6.

---

### Step 4.4 — Verify the image is in ECR

```bash
aws ecr describe-images \
  --repository-name auth-service \
  --region us-east-1 \
  --query 'sort_by(imageDetails, &imagePushedAt)[-1].imageTags' \
  --output table
```

You should see your `sha-abc1234` tag in the output.

---

## Part 5 — Watch ArgoCD deploy to DEV

The moment Job 2 committed to `zen-gitops-lab1`, ArgoCD began a sync. It polls the repo every ~3 minutes.

### Step 5.1 — Check the ArgoCD app

```bash
argocd login <ARGOCD-URL> --username admin --password <PASSWORD>
argocd app get auth-service-dev
```

Or open the ArgoCD UI in your browser (URL from instructor).

You should see:
```
Sync Status:   Synced
Health Status: Progressing  →  Healthy
```

### Step 5.2 — Watch the pod come up

```bash
kubectl get pods -n dev -w
```

Expected output:
```
NAME                           READY   STATUS              RESTARTS   AGE
auth-service-7d9f8c-xxxx       0/1     ContainerCreating   0          5s
auth-service-7d9f8c-xxxx       0/1     Running             0          20s
auth-service-7d9f8c-xxxx       1/1     Running             0          55s
```

The pod reaches `1/1 Ready` once the readiness probe passes. Spring Boot takes ~30 seconds to start.

Press **Ctrl+C** to exit the watch.

### Step 5.3 — Confirm which image is running

```bash
kubectl get pods -n dev -l app.kubernetes.io/name=auth-service \
  -o jsonpath='{.items[0].spec.containers[0].image}'
```

Expected: `<ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/auth-service:sha-abc1234`

### Step 5.4 — Hit the health endpoint

```bash
kubectl port-forward -n dev deploy/auth-service 8081:8081 &
curl http://localhost:8081/actuator/health
```

Expected: `{"status":"UP"}`

```bash
kill %1   # stop the port-forward
```

---

## Part 6 — ArgoCD self-heal demo

This is the key GitOps concept: **the cluster always converges back to what git says**.

If someone makes a manual change on the cluster — deletes a deployment, scales replicas, patches a configmap — ArgoCD detects the drift and reverts it.

### Step 6.1 — Enable selfHeal on the ArgoCD app

The app was created with `selfHeal` disabled. Enable it now:

```bash
argocd app set auth-service-dev --self-heal
```

Verify:
```bash
argocd app get auth-service-dev | grep -A5 "Sync Policy"
# Auto-Sync: enabled (Prune=true, SelfHeal=true)
```

### Step 6.2 — Delete the deployment

```bash
kubectl delete deployment auth-service -n dev
```

Watch what happens:

```bash
kubectl get pods -n dev -w
```

The running pod terminates:
```
auth-service-7d9f8c-xxxx   1/1   Running       0   15m
auth-service-7d9f8c-xxxx   1/1   Terminating   0   15m
auth-service-7d9f8c-xxxx   0/1   Terminating   0   15m
```

### Step 6.3 — Watch ArgoCD detect and fix it

```bash
watch argocd app get auth-service-dev
```

Within ~3 minutes:

1. ArgoCD detects the deployment is missing → marks app `OutOfSync / Missing`
2. ArgoCD applies git state back to cluster → `Syncing`
3. Deployment recreated → pod starts → `Synced / Healthy`

```
auth-service-8e2a1f-yyyy   0/1   ContainerCreating   0   5s
auth-service-8e2a1f-yyyy   1/1   Running             0   50s
```

Press **Ctrl+C** to exit the watch.

**The key insight:** You deleted a Kubernetes resource. ArgoCD brought it back — automatically, without anyone intervening, without any alert firing. Git is the single source of truth. The cluster must match git, always.

### Step 6.4 — Try changing replica count directly

```bash
kubectl scale deployment auth-service -n dev --replicas=3
```

Watch it get reverted:

```bash
watch kubectl get deployment auth-service -n dev -o wide
```

Within 3 minutes, replicas go from `3` → `1` (what the values file says).

---

## Part 7 — QA promotion

The QA PR was opened automatically when the build finished. Merging it deploys the **same image** to QA — no rebuild.

### Step 7.1 — Review and merge the QA PR

1. Open your `zen-gitops-lab1` fork → **Pull requests**
2. Open `promote(qa): auth-service → sha-abc1234`
3. Look at the diff — it changes **only** `image.tag` in `envs/qa/values-auth-service.yaml`
4. Merge it

### Step 7.2 — Watch ArgoCD deploy to QA

```bash
argocd app get auth-service-qa
kubectl get pods -n qa -w
```

Same `sha-abc1234` image, different namespace, different profile (`qa`), different DB host. No rebuild occurred.

---

## Complete flow — what you just built

```
your laptop
    │
    │  git push feat-*
    ▼
GitHub Actions PR check
    tests + SAST (~5 min, no Docker)
    │
    │  PR merged → develop
    ▼
GitHub Actions full CI
    mvn verify → CodeQL → Semgrep
    Docker build → Trivy scan
    ECR push → sha-abc1234
    Cosign sign → Rekor log
    │
    ├──► zen-gitops-lab1: envs/dev/values-auth-service.yaml
    │         image.tag: sha-abc1234
    │              │
    │              ▼
    │         ArgoCD auto-sync DEV
    │         helm template → kubectl apply
    │         DEV pod: sha-abc1234 ✓
    │
    └──► zen-gitops-lab1 PR: envs/qa/values-auth-service.yaml
              (merge when QA team approves)
                   │
                   ▼
              ArgoCD auto-sync QA
              same image, new namespace ✓
```

CI never talks to Kubernetes. CI talks to git. Kubernetes gets its orders from git.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Job fails: `Error assuming role` | `AWS_ACCOUNT_ID` secret wrong or OIDC trust policy mismatch | Confirm the 12-digit account ID and that the repo name in the trust policy matches your fork exactly |
| Job fails: `denied: Your authorization token has expired` | ECR login failed | Double-check `AWS_ACCOUNT_ID` secret |
| `deploy-dev` fails: `Authentication failed for https://github.com/...` | `GITOPS_TOKEN` expired or scoped to wrong repo | Regenerate PAT with Contents + Pull requests write on your zen-gitops-lab1 fork |
| Maven tests fail: `Connection refused` to DB | PostgreSQL not ready or `needs-database` flag missing | Check `ci-auth-service.yml` has `needs-database: true` passed to the reusable workflow |
| Pod stuck in `CrashLoopBackOff` | Spring Boot cannot connect to RDS | `kubectl logs -n dev deploy/auth-service` — check `DB_HOST` in your values file |
| Pod stuck in `ImagePullBackOff` | ECR pull auth issue | Check the service account's IAM role annotation; verify image tag exists in ECR |
| ArgoCD shows `OutOfSync` but not healing | `selfHeal` not enabled | Run `argocd app set auth-service-dev --self-heal` |
| ArgoCD app still pointing at `DPP-2026/zen-gitops-lab1` | App repoURL not updated to your fork | Notify instructor — the app needs to be re-pointed to `https://github.com/<YOUR-USERNAME>/zen-gitops-lab1.git` |
| `argocd: command not found` | argocd CLI not installed | `brew install argocd` (Mac) or see https://argo-cd.readthedocs.io/en/stable/cli_installation/ |
