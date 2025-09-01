TickBoard GitOps (Kustomize + Argo CD)

Structure
- base: Namespace, Mongo (ephemeral), gin-api, frontend, Ingress
- overlays/dev: Dev overrides, Ingress host patch, sample Secret
- argocd: Example Application manifest pointing to this repo
- apps: Argo CD AppProjects and an ApplicationSet example
- stacks/cluster: Cluster-scoped bits (e.g., External Secrets ClusterSecretStore)

Quick Start
1) Build & push images
   - harbor.czhuang.dev/library/tickboard-gin-api:latest
   - harbor.czhuang.dev/library/tickboard-frontend:latest

2) Prepare secrets
   - Copy overlays/dev/secret.sample.yaml to overlays/dev/secret.yaml
   - Edit values (JWT_SECRET, DB_NAME, MONGO_URI, etc.)
   - Consider Sealed Secrets or SOPS for real environments

3) Install Argo CD on cluster (once)
   - kubectl create namespace argocd
   - kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

4) Create Application
   - Update repoURL in argocd/app-dev.yaml
   - kubectl apply -f argocd/app-dev.yaml

Notes
- Ingress host in overlays/dev/ingress-patch.yaml defaults to tickboard.dev.example.com; set a real host and DNS.
- Mongo in base is ephemeral (emptyDir); replace with StatefulSet + PVC or use managed DB in prod.
- Frontend calls "/api" via Ingress path routing; no REACT_APP_* override needed.

Prerequisites
- kubectl: Cluster access with appropriate permissions
- kustomize: v4+ (kubectl -k is fine if it supports Kustomize v4)
- Argo CD: v2.5+ recommended
- Container registry: Access to Harbor project and credentials if required

Local Validate & Render
- Validate manifests with kubeconform:
  - See GitHub Action `workflows/gitops-validate.yaml` for the exact invocation.
- Dry-run render overlay:
  - kustomize build gitops/stacks/tickboard/overlays/dev
- Optional kubectl diff/apply (for quick checks outside Argo CD):
  - kubectl diff -k gitops/stacks/tickboard/overlays/dev
  - kubectl apply -k gitops/stacks/tickboard/overlays/dev

Images & Overlays
- Base images use GHCR placeholders to keep base generic.
- The dev overlay rewrites images to Harbor using kustomize `images`:
  - ghcr.io/OWNER/tickboard-gin-api → harbor.czhuang.dev/library/tickboard-gin-api:latest
  - ghcr.io/OWNER/tickboard-frontend → harbor.czhuang.dev/library/tickboard-frontend:latest
- CI can update the overlay tag (see `workflows/ci-cd.yaml`).

Secrets & Registry Auth
- Application secrets: place real values in `overlays/dev/secret.yaml` (do not commit). For production, prefer Sealed Secrets or SOPS.
- Harbor auth (if your cluster needs it to pull images):
  - Create a docker-registry Secret in the `tickboard` namespace:
    - kubectl -n tickboard create secret docker-registry harbor-creds \
      --docker-server=harbor.czhuang.dev \
      --docker-username=$HARBOR_USERNAME \
      --docker-password=$HARBOR_PASSWORD
  - Reference it via `imagePullSecrets` in an overlay patch (Deployment or default ServiceAccount).

Ingress Controller
- Base assumes AWS ALB (`ingressClassName: alb` + annotations).
- If using another controller (e.g., NGINX Ingress):
  - Patch the Ingress in your overlay to set `ingressClassName: nginx` and adjust annotations accordingly.

MongoDB Persistence
- The base Mongo Deployment uses `emptyDir` (ephemeral). For anything beyond dev:
  - Replace with a StatefulSet + PVC, or
  - Use a managed MongoDB service and point `MONGO_URI` to it.

Argo CD Models
- Single app (recommended for now): `argocd/app-dev.yaml` targets the dev overlay.
- ApplicationSet example (`gitops/apps/workloads-appset.yaml`) currently scans `gitops/stacks/*` (base).
  - If you want ApplicationSet to deploy overlays, change generator directories to `gitops/stacks/*/overlays/*` and adjust app naming and destination namespace accordingly.

CI/CD Overview
- `workflows/gitops-validate.yaml`: Validates all `gitops/**` manifests with kubeconform.
- `workflows/ci-cd.yaml`: Optional image build/push to Harbor and auto-bump `overlays/dev` image tag.
  - Requires GitHub secrets: `HARBOR_REGISTRY`, `HARBOR_PROJECT`, `HARBOR_USERNAME`, `HARBOR_PASSWORD`, and `AWS_GHA_ROLE_ARN` if using AWS OIDC.

Troubleshooting
- Ingress 404/host mismatch:
  - Ensure `overlays/dev/ingress-patch.yaml` host matches your DNS and the Ingress controller is healthy.
- Image pull errors:
  - Confirm Harbor Secret exists and is referenced via `imagePullSecrets`; ensure the image/tag exists in Harbor.
- Argo CD out-of-sync:
  - Check project permissions and repoURL correctness; ensure `syncOptions: CreateNamespace=true` for new namespaces.
- Mongo connection issues:
  - Verify `MONGO_URI` host matches the in-cluster Service (mongo.tickboard.svc.cluster.local) or your external endpoint.

