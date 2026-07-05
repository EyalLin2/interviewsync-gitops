# interviewsync-gitops

ArgoCD desired state repository for InterviewSync.

This repo is the "source of truth" for what should be running in each environment.
ArgoCD watches this repo and automatically syncs any changes to the cluster.
GitHub Actions CI writes to this repo (updates the image tag) to trigger deployments.

## Structure

```
apps/
  staging/
    backend-values.yaml    ← Helm values for backend (staging)
    frontend-values.yaml   ← Helm values for frontend (staging)
    postgres-values.yaml   ← Helm values for PostgreSQL (staging)
  production/
    ...                    ← Same structure for production

argocd/
  staging/
    app-backend.yaml       ← ArgoCD Application CR → points to InterviewSync/backend/charts
    app-frontend.yaml      ← ArgoCD Application CR → points to InterviewSync/frontend/charts
    app-postgres.yaml      ← ArgoCD Application CR → uses bitnami/postgresql chart
  production/
    ...
```

## Deployment flow

```
Developer pushes to main
        │
        ▼
GitHub Actions (deploy.yml)
  1. Builds images → pushes to ECR
  2. Updates apps/staging/backend-values.yaml  ← image.tag = git SHA
  3. Updates apps/staging/frontend-values.yaml ← image.tag = git SHA
  4. Commits + pushes to this repo
        │
        ▼
ArgoCD detects change in this repo (polls every 3 minutes)
  → runs helm upgrade with new values
  → DB init Job runs first (Helm pre-upgrade hook)
  → new pods roll out
  → old pods terminate
```

## First-time setup

After running `terraform apply` in `interviewsync-infra/environments/staging/`:

```bash
# 1. Get kubeconfig
aws eks update-kubeconfig --region us-east-1 --name staging-interviewsync

# 2. Apply all staging ArgoCD Application manifests
# Replace <YOUR_USERNAME> in the yaml files with your GitHub username first
kubectl apply -f argocd/staging/

# 3. Watch ArgoCD sync everything
kubectl get applications -n argocd -w
```

## Updating image tags manually

Normally done by CI, but can be done manually:

```bash
# Edit the tag in the relevant values file, then commit+push
sed -i 's/tag: .*/tag: "abc1234"/' apps/staging/backend-values.yaml
git add -A && git commit -m "manual: update backend to abc1234" && git push
# ArgoCD picks this up within ~3 minutes
```

## Accessing ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open http://localhost:8080
# Username: admin
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
