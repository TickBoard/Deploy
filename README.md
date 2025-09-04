TickBoard Deploy (GitOps)

Overview
- Declarative GitOps manifests for TickBoard using Kustomize and Argo CD.
- Base provides generic Kubernetes objects; overlays adjust per environment (e.g., dev).
- Single CI workflow validates and deploys to EKS via Argo CD on push to `main`.

Repository Structure
- `gitops/`
  - `apps/`: Argo CD AppProjects and an ApplicationSet example
  - `stacks/`
    - `tickboard/`
      - `base/`: Namespace, Mongo (ephemeral), `gin-api`, `frontend`, Ingress
      - `overlays/dev/`: Dev overrides (ServiceAccount imagePullSecret, sample Secret)
      - `argocd/app-dev.yaml`: Example Argo CD Application targeting the dev overlay
- `.github/workflows/`: GitHub Actions (CI/CD)

Quick Start
1) Prerequisites
   - kubectl + access to your cluster
   - Kustomize v4+ (or kubectl `-k` with v4 support)
   - Argo CD installed on the cluster
   - Harbor registry credentials (if required)

2) Prepare images
   - Build and push:
     - `harbor.czhuang.dev/tickboard/gin-api:latest`
     - `harbor.czhuang.dev/tickboard/frontend:latest`

3) Prepare secrets (dev)
   - Copy `gitops/stacks/tickboard/overlays/dev/secret.sample.yaml` to `secret.yaml`
   - Edit `JWT_SECRET`, `DB_NAME`, `MONGO_URI`, `MONGO_ROOT_*` (Do not commit real secrets.)

4) Deploy via Argo CD (example)
   - If deploy lives inside a mono‑repo, update Argo CD `path` values accordingly.
   - Edit `repoURL` to your Git repo (accessible by Argo CD)
   - Apply Application: `kubectl apply -f gitops/stacks/tickboard/argocd/app-dev.yaml`

Ingress & Controllers
- Assumes AWS ALB (`ingressClassName: alb` + ALB annotations) on the Ingress.
  - Target group health checks are set per Service via annotations:
    - `gin-api` → `/api/health`
    - `frontend` → `/`
  - Ensure AWS Load Balancer Controller is installed and DNS points to the ALB.

Images & Overlays
- Base images use GHCR placeholders; the dev overlay rewrites to Harbor via `images`:
  - `ghcr.io/OWNER/tickboard-gin-api` → `harbor.czhuang.dev/tickboard/gin-api:latest`
  - `ghcr.io/OWNER/tickboard-frontend` → `harbor.czhuang.dev/tickboard/frontend:latest`

Validation & Troubleshooting
- Validate manifests locally: `kustomize build gitops/stacks/tickboard/overlays/dev`
- Common issues
  - Ingress 404: Verify host and DNS
  - Image pull: Ensure Harbor image exists and `imagePullSecrets` are configured
    - Preferred: `serviceaccount-patch.yaml` adds `harbor-pull-secret` to the default SA
    - Create the pull secret:
      `kubectl -n tickboard create secret docker-registry harbor-pull-secret --docker-server=harbor.czhuang.dev --docker-username=<user> --docker-password=<pwd> --docker-email=<email>`
  - Mongo DB: Base uses `emptyDir`; for persistence, use a StatefulSet or managed DB

CI/CD
- `.github/workflows/gitops-ci.yaml`: Validates Kustomize with kubeconform on PR/push; on push to `main` (and validation success) deploys via Argo CD to EKS.

Notes
- Deploy from overlays (not base) so image/host overrides apply (e.g., `overlays/dev`).
- `gitops/apps/workloads-appset.yaml` is an example scanning overlays; adjust as needed.
