TickBoard Deploy (GitOps)

Overview
- Declarative GitOps manifests for TickBoard using Kustomize and Argo CD.
- Base provides generic Kubernetes objects; overlays adjust per environment (e.g., dev).
- CI includes a manifest validator; optional image build workflow is included as a template.

Repository Structure
- `gitops/`
  - `apps/`: Argo CD AppProjects and an ApplicationSet example
  - `stacks/`
    - `tickboard/`
      - `base/`: Namespace, Mongo (ephemeral), `gin-api`, `frontend`, Ingress
      - `overlays/dev/`: Dev overrides (Ingress host patch, sample Secret)
      - `argocd/app-dev.yaml`: Example Argo CD Application targeting the dev overlay
    - `cluster/`: Cluster-scoped bits (e.g., External Secrets `ClusterSecretStore`)
- `workflows/`: GitHub Actions (CI/CD, manifest validation)

Quick Start
1) Prerequisites
   - kubectl + access to your cluster
   - Kustomize v4+ (or kubectl `-k` with v4 support)
   - Argo CD installed on the cluster
   - Harbor registry credentials (if required)

2) Prepare images
   - Build and push:
     - `harbor.czhuang.dev/library/tickboard-gin-api:latest`
     - `harbor.czhuang.dev/library/tickboard-frontend:latest`

3) Prepare secrets (dev)
   - Copy `gitops/stacks/tickboard/overlays/dev/secret.sample.yaml` to `secret.yaml`
   - Edit `JWT_SECRET`, `DB_NAME`, `MONGO_URI`, etc. (Do not commit real secrets.)

4) Deploy via Argo CD
   - Edit `gitops/stacks/tickboard/argocd/app-dev.yaml` and set `repoURL` to this repo
   - `kubectl apply -f gitops/stacks/tickboard/argocd/app-dev.yaml`

Ingress & Controllers
- Base assumes AWS ALB (`ingressClassName: alb` + ALB annotations). If using another controller (e.g., NGINX), patch the Ingress in your overlay accordingly.

Images & Overlays
- Base images use GHCR placeholders; the dev overlay rewrites to Harbor via the `images` section:
  - `ghcr.io/OWNER/tickboard-gin-api` → `harbor.czhuang.dev/library/tickboard-gin-api:latest`
  - `ghcr.io/OWNER/tickboard-frontend` → `harbor.czhuang.dev/library/tickboard-frontend:latest`

Validation & Troubleshooting
- Validate manifests locally: `kustomize build gitops/stacks/tickboard/overlays/dev`
- CI manifest validation: see `workflows/gitops-validate.yaml`
- Common issues
  - Ingress 404: Verify host in `overlays/dev/ingress-patch.yaml` and DNS
  - Image pull: Ensure Harbor image exists and `imagePullSecrets` are configured if needed
  - Mongo DB: Base uses `emptyDir`; for persistence, use a StatefulSet or managed DB

CI/CD
- `workflows/gitops-validate.yaml`: kubeconform validation for `gitops/**`
- `workflows/ci-cd.yaml`: Example image build/push to Harbor and auto-bump overlay tag (requires secrets). Optional if this repo does not contain app code.

Notes
- Deploy from overlays (not base) so image/host overrides apply (e.g., `overlays/dev`).
- `gitops/apps/workloads-appset.yaml` is an example; adjust it to scan overlay paths if you want ApplicationSet to manage environment overlays.

