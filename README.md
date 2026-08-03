# From DevOps to Platform Engineering: Master Backstage & IDPs

Hands-on project workspace for the [Udemy course](https://leidos.udemy.com/course/from-devops-to-platform-engineering-master-backstage-idps/learn/lecture/49124433#overview). Covers Docker, Kubernetes, ArgoCD, GitHub Actions, and Backstage IDP.

## Progress

### Foundation
- ✅ Flask API (`src/app.py`)
- ✅ Dockerfile
- ✅ Helm chart (`charts/python-app/`)
- ⬜ Raw K8s manifests (`k8s/`)

### CI/CD
- ⬜ GitHub Actions pipeline

### GitOps / ArgoCD
- ⬜ ArgoCD Application manifest
- ⬜ ArgoCD sync configured

### Backstage IDP
- ⬜ Backstage instance deployed
- ⬜ `catalog-info.yaml` (service registered)
- ⬜ TechDocs site
- ⬜ Software Template

### Production Deployment
- ⬜ Backstage deployed to Kubernetes

## Quick Reference

```bash
# Run locally
pip install -r requirements.txt
python src/app.py

# Build and run Docker image
docker build -t cdiamond/python-app:v2 .
docker run -p 5000:5000 cdiamond/python-app:v2

# Helm install / upgrade
helm upgrade --install python-app charts/python-app
helm upgrade --install python-app charts/python-app --set image.tag=v3
```
