# 🚀 QuickHaul: Full Infrastructure Setup Guide

This guide provides step-by-step instructions for setting up the QuickHaul ecosystem on an **AWS EC2 Instance**, including the CI/CD tools and the GitOps engine.

---

## 🏗️ 1. EC2 Instance Preparation

### Recommended Specs:
-   **Instance Type**: `t3.medium` (minimum 4GB RAM required for SonarQube).
-   **OS**: Ubuntu 22.04 LTS.
-   **Storage**: 30GB+ SSD.

### Security Group Configuration (Inbound Rules):
| Port | Protocol | Purpose |
| :--- | :--- | :--- |
| 22 | TCP | SSH Access |
| 80 / 443 | TCP | HTTP/HTTPS (ArgoCD / Web Apps) |
| 9000 | TCP | SonarQube UI |
| 6443 | TCP | Kubernetes API (if accessing remotely) |

### Base Installation:
```bash
# Update System
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Lightweight Kubernetes (k3s)
curl -sfL https://get.k3s.io | sh -
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```

---

## 🔍 2. SonarQube (SAST) Setup

SonarQube is used for Static Application Security Testing.

### Run with Docker:
```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community
```
1.  Access via `http://<EC2-IP>:9000` (Default login: `admin/admin`).
2.  **Generate Token**: My Account > Security > Generate Token.
3.  **Create Project**: Manual > Name it `quickhaul`.
4.  Add `SONAR_TOKEN` and `SONAR_URL` (http://<EC2-IP>:9000) to GitHub Secrets.

---

## 🛡️ 3. SCA & Container Security

### Snyk (Dependency Scan):
1.  Sign up at [snyk.io](https://snyk.io).
2.  Go to **Account Settings** and copy your **API Token**.
3.  Add it to GitHub Secrets as `SNYK_TOKEN`.

### Trivy (Image Scan):
Trivy is integrated into the `_ci-template.yml`. It doesn't require a standalone server but runs as a step in the GitHub Action runner to scan the built image before pushing to GHCR.

---

## 📦 4. GitHub Container Registry (GHCR)

The CI pipeline is now configured to push images to **GHCR** (`ghcr.io`).

1.  **Authentication**: The workflow uses `secrets.GITHUB_TOKEN` automatically.
2.  **Visibility**: By default, GHCR images are private. Ensure you grant the `GITHUB_TOKEN` "Read and Write" permissions in your repository settings (Settings > Actions > General > Workflow permissions).

---

## 🎡 5. ArgoCD Setup (GitOps)

ArgoCD synchronizes your cluster state with the [`quickhaul-config`](https://github.com/QuickHaulTransits/quickhaul-config) repository.

### Installation:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the UI (Forward port or use LoadBalancer)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Connect Private Repository:
1.  Login to ArgoCD UI.
2.  Go to **Settings > Repositories > Connect Repo**.
3.  Use **HTTPS** with a GitHub **Personal Access Token (PAT)** or **SSH** with a Deploy Key.
4.  Repo URL: `https://github.com/QuickHaulTransits/quickhaul-config.git`.

### Create Application:
1.  **Application Name**: `quickhaul-microservices`.
2.  **Project**: `default`.
3.  **Sync Policy**: `Automatic`.
4.  **Source**: Path `argocd-apps/`, Repo URL above.
5.  **Destination**: Cluster `https://kubernetes.default.svc`, Namespace `quickhaul-dev`.

---

## 🔄 How the Change Flow Works

1.  **Developer Pushes Code**: Triggers the CI workflow.
2.  **CI Build**:
    *   `SonarQube` scans code quality.
    *   `Snyk` scans dependencies.
    *   `Docker` builds the image and tags it (e.g., `v1.0.5`).
    *   `Trivy` scans the image.
    *   Image is pushed to `ghcr.io`.
3.  **CD Trigger**:
    *   `_cd-template.yml` clones `quickhaul-config`.
    *   It updates `helm-charts/microservices/<service>/values.yaml` with the new `image.tag: v1.0.5`.
    *   It commits and pushes back to `quickhaul-config`.
4.  **ArgoCD Sync**:
    *   ArgoCD detects the commit in `quickhaul-config`.
    *   It compares the cluster state with the repo.
    *   It triggers a `RollingUpdate` or `Argo Rollout` to deploy the new image version.

---

## 📞 Troubleshooting
-   **SonarQube Memory**: If it crashes, ensure `vm.max_map_count` is set:
    `sudo sysctl -w vm.max_map_count=262144`
-   **ArgoCD Sync Fail**: Check logs with `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-controller`.
-   **GHCR Access**: If K8s can't pull images, create an imagePullSecret:
    ```bash
    kubectl create secret docker-registry ghcr-secret \
      --docker-server=ghcr.io \
      --docker-username=<GITHUB-USER> \
      --docker-password=<PAT>
    ```
