# TickBoard Deploy (GitOps)

Declarative GitOps manifests for TickBoard using Kustomize and Argo CD, with GitHub Actions for validation and automated deployment.

## Features
- GitOps: All config is versioned in Git and synced by Argo CD.
- Environment overlays: Kustomize base + overlays (currently dev).
- CI validation: Kustomize build + kubeconform schema checks on PR/push.
- CD deploy: On push to `main`, applies Root App + ApplicationSet into EKS.
- Release flow: Manual workflow to bump overlay image tags via PR.

## Repository Layout
- `gitops/`
  - `apps/`: Argo CD AppProjects, Root Application, ApplicationSet
    - `projects/`: `cluster` and `apps` AppProjects
    - `root-app.yaml`: Root Application (recursively applies `gitops/apps` content)
    - `workloads-appset.yaml`: Scans overlays to create app instances
  - `stacks/tickboard/`
    - `base/`: Namespace, Mongo (ephemeral), `gin-api`, `frontend`, Ingress
    - `overlays/dev/`: Dev overrides (Harbor image rewrite, SA imagePullSecret, Secret example)
    - `argocd/app-dev.yaml`: Single Application example targeting the dev overlay
- `.github/workflows/`: CI validation and deploy, release, and Dependabot auto-merge
- `.gitignore`: Lists `gitops/stacks/tickboard/overlays/dev/secret.yaml`

## Prerequisites
- Kubernetes cluster (EKS) and `kubectl` access
- Kustomize v4+ (or `kubectl -k` with v4 support)
- Argo CD installed in the cluster
- AWS CLI (CI sets EKS context)
- If using AWS ALB: AWS Load Balancer Controller installed
- Container registry: Harbor project and credentials

## Quick Start
### 1) Install Argo CD (one-time on cluster)
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2) Prepare container images
Push to Harbor:
- `harbor.czhuang.dev/tickboard/gin-api:latest`
- `harbor.czhuang.dev/tickboard/frontend:latest`

### 3) Configure secrets (dev)
A sample file is included at `gitops/stacks/tickboard/overlays/dev/secret.yaml` with example values. Update `JWT_SECRET`, `DB_NAME`, `MONGO_URI`, `MONGO_ROOT_*` and apply locally:
```bash
kubectl apply -f gitops/stacks/tickboard/overlays/dev/secret.yaml
```
Note: The path is listed in `.gitignore` for safety; however, since a sample file is committed at that path, Git will track it. Do not commit real secrets—treat the committed file as example values only, or use Sealed Secrets/SOPS for production.

### 4) Deploy (choose one)
- Option A: Single Application (good for a quick dev check)
```bash
kubectl apply -f gitops/stacks/tickboard/argocd/app-dev.yaml
```
- Option B: GitHub Actions + Root App + ApplicationSet (recommended)
  - Configure repo Variables/Secrets (see CI/CD)
  - Merge to `main` to trigger validate and deploy

## CI/CD (GitHub Actions)
- Validation and deploy: `.github/workflows/gitops-ci.yaml`
  - Triggers: PR and push (limited to changes under `gitops/**`)
  - Validation: installs `kustomize` and `kubeconform`; runs `kustomize build | kubeconform -strict -ignore-missing-schemas -summary -` for overlays and bases
  - Deploy (only on push to `main` + validation success):
    - Assumes AWS role via OIDC, sets EKS context
    - Ensures `tickboard` namespace and (optionally) Harbor pull secret
    - Applies: `gitops/apps/projects/*.yaml`, `gitops/apps/root-app.yaml`, `gitops/apps/workloads-appset.yaml`

Required GitHub repo configuration:
- Variables
  - `AWS_REGION` (e.g., `ap-northeast-1`)
  - `EKS_CLUSTER_NAME` (target EKS cluster)
- Secrets
  - `AWS_GHA_ROLE_ARN` (IAM Role ARN for OIDC)
  - Optional Harbor credentials (skipped if absent):
    - `HARBOR_REGISTRY` (default `harbor.czhuang.dev`)
    - `HARBOR_USERNAME`
    - `HARBOR_PASSWORD`
    - `HARBOR_EMAIL` (default `ci@example.com`)

## Release Flow (bump overlay image tags)
- Workflow: `.github/workflows/gitops-release.yaml` (manual `workflow_dispatch`)
- Action: Updates `images.*.newTag` in `gitops/stacks/tickboard/overlays/dev/kustomization.yaml` and opens a PR
- Inputs:
  - `tag`: set both API/FE to the same tag (optional)
  - `api_tag`, `fe_tag`: set tags individually (ignored if `tag` is provided)

## Dependency Management
- Dependabot: `.github/dependabot.yml` checks GitHub Actions monthly
- Auto-merge: `.github/workflows/dependabot-auto-merge.yaml` enables auto-merge for Dependabot PRs with patch/minor updates

## Kustomize Overlays & Images
- Base images use GHCR placeholders: `ghcr.io/OWNER/tickboard-*`
- The dev overlay rewrites images and sets tags:
  - `harbor.czhuang.dev/tickboard/gin-api:<tag>`
  - `harbor.czhuang.dev/tickboard/frontend:<tag>`
- Registry pull secret (via ServiceAccount patch recommended)
  - Overlay includes `serviceaccount-patch.yaml` to attach `harbor-pull-secret` to the default SA
  - Create secret example:
```bash
kubectl -n tickboard create secret docker-registry harbor-pull-secret \
  --docker-server=harbor.czhuang.dev \
  --docker-username=<user> \
  --docker-password=<pwd> \
  --docker-email=<email>
```

## Ingress & Domain
- Ingress (`gitops/stacks/tickboard/base/ingress.yaml`) assumes AWS ALB (`ingressClassName: alb`)
- Default host: `tickboard.czhuang.dev`; change to your domain and configure DNS to the ALB
- Using NGINX Ingress? Patch `ingressClassName: nginx` and adjust annotations

## MongoDB Notes
- The base Mongo `Deployment` uses `emptyDir` (ephemeral; not for production)
- For production:
  - Switch to StatefulSet + PVC, or
  - Use a managed MongoDB and point `MONGO_URI` to it

## Local Validation
- Render overlay:
```bash
kustomize build gitops/stacks/tickboard/overlays/dev
```
- Schema validation (requires kubeconform):
```bash
kustomize build gitops/stacks/tickboard/overlays/dev | kubeconform -strict -ignore-missing-schemas -summary -
```
- Diff/apply without Argo CD:
```bash
kubectl diff -k gitops/stacks/tickboard/overlays/dev
kubectl apply -k gitops/stacks/tickboard/overlays/dev
```

## Troubleshooting
- Ingress 404/host mismatch: check Ingress host, DNS, and ALB/Ingress Controller health
- Image pull errors: ensure Harbor secret is present and referenced by `imagePullSecrets`; verify image:tag exists
- Argo CD out-of-sync: verify `repoURL`/`path` and `syncOptions: CreateNamespace=true`
- Mongo connection: verify `MONGO_URI`; for in-cluster, use `mongo.tickboard.svc.cluster.local`

## Security
- Do not commit real secrets.
- Prefer Sealed Secrets or SOPS for production secrets.

## Notes
- If you fork or rename the repo, update `repoURL` and `path` in:
  - `gitops/apps/root-app.yaml`
  - `gitops/apps/workloads-appset.yaml`
  - `gitops/stacks/tickboard/argocd/app-dev.yaml`

