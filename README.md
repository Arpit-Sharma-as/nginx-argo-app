# Ì∫Ä Argo CD with Image Updater using Helm on Minikube (Docker Driver)

This project demonstrates how to deploy **Argo CD** and **Argo CD Image Updater** using **Helm**, configure GitHub repo credentials, and enable automatic Docker image updates using annotations.

---

## Ì∑± Prerequisites

- Docker installed and running
- Minikube (v1.25+)
- kubectl
- Helm (v3.8+)
- GitHub account & repo access with write permissions
- A public Docker image (e.g., `docker.io/bucket23/nginx`)

---

## Ì∞≥ Start Minikube with Docker Driver

```bash
minikube start --driver=docker
```

---

## Ì≥¶ Install Argo CD using Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace
kubectl create ns argocd

# Install Argo CD
helm install argocd argo/argo-cd -n argocd   --set server.service.type=NodePort   --set configs.params."server\.insecure"=true
```

### ÔøΩÔøΩ Get Argo CD Admin Password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

---

## Ìºê Access Argo CD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

Open: http://localhost:8080  
Login: `admin` / `<retrieved-password>`

---

## Ì¥Å Install Argo CD Image Updater via Helm

```bash
helm install argocd-image-updater argo/argocd-image-updater -n argocd   --set argocd.appSync=true   --set config.updateStrategy=latest   --set metrics.enabled=true
```

---

## Ì¥í Configure GitHub Repo Credentials (for Image Updater Write-Back)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds
stringData:
  url: https://github.com/<YOUR_USER_NAME>/nginx-argo-app.git
  username: YOUR_USER_NAME
  password: YOUR_PAT_TOKEN
type: Opaque
```

Save the file as `git-credentials.yaml` Apply the creds file:

```bash
kubectl apply -f git-credentials.yaml
```

**Note:** Your GitHub PAT must have `repo` scope to allow commits.

---

## Ì≥Ñ Sample Argo CD Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: docker.io/bucket23/nginx
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/update-strategy: latest
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/<repo>.git
    targetRevision: master
    path: nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## ‚öôÔ∏è How Image Updater Works

- On detecting a new image tag, it creates a file:
  
  ```
  <application-path>/.argocd-source-<app-name>.yaml
  ```

  Example content:

  ```yaml
  kustomize:
    images:
    - docker.io/bucket23/nginx:1.22
  ```

- Argo CD merges this file with your Kustomize config and syncs the update.

---

## Ìª†Ô∏è Troubleshooting

- ‚ùó `Could not get creds for repo`: Ensure GitHub PAT is correct and repo is public/private with access.
- ‚ùó `skipping app of type 'Directory'`: Ensure `path` and repo are structured correctly with `kustomization.yaml`.
- ‚ùó `failed to generate manifest...`: Make sure `kustomization.yaml` has correct structure and values.

---

## ‚úÖ Final Validation

1. Push a new Docker image tag (e.g., `1.22`)
2. Argo CD Image Updater detects and commits `.argocd-source-*.yaml`
3. Argo CD syncs automatically to update the deployment

---

## Ì≥ö References

- [Argo CD](https://argo-cd.readthedocs.io/)
- [Argo CD Image Updater](https://argocd-image-updater.readthedocs.io/)
- [Helm Chart](https://github.com/argoproj/argo-helm)


