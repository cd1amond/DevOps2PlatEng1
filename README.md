# From DevOps to Platform Engineering: Master Backstage & IDPs

Hands-on project workspace for the [Udemy course](https://leidos.udemy.com/course/from-devops-to-platform-engineering-master-backstage-idps/learn/lecture/49124433#overview). Covers Docker, Kubernetes, ArgoCD, GitHub Actions, and Backstage IDP.

## Progress

### Section 1: Introduction ✅
Conceptual overview — Platform Engineering, IDPs, and Backstage. No artifacts.

### Section 2: Level Up Your DevOps Skills

**Python / Docker**
- ✅ Flask API (`src/app.py`)
- ✅ Dockerfile
- ✅ Image pushed to Docker Hub registry (`cdiamond/python-app`)

**Kubernetes**
- ✅ Raw K8s manifests (`k8s/deploy.yaml`, `k8s/service.yaml`, `k8s/ingress.yaml`)
- ✅ Local cluster running (kind/Docker)
- ✅ Ingress controller configured (ingress-nginx v1.11.3, pulled directly from `registry.k8s.io` after fixing the node's CA trust store — see CLAUDE.md)
- ✅ `python-app` Ingress (`k8s/ingress.yaml`, host `python-app.test.com`) — routes correctly, verified from inside the Rancher Desktop VM. Not reachable from the Mac browser: `devops-course` was created without `extraPortMappings`, so the NodePort isn't published to the host. Would need cluster recreation to fix — see CLAUDE.md.

**Helm**
- ✅ Helm chart (`charts/python-app/`)
- ✅ Chart deployed to local cluster

**GitOps / ArgoCD**
- ✅ ArgoCD deployed to cluster (v2.13.6, via kubectl apply + Harbor proxy cache)
- ⬜ ArgoCD Application manifest
- ⬜ App synced via ArgoCD

**CI — GitHub Actions**
- ✅ CI workflow (build + push image with dynamic tags)
- ✅ GitHub secrets configured (Docker credentials)

**CD — GitHub Actions**
- ⬜ Self-hosted runner configured  ← starting here (lecture 48)
- ⬜ CD workflow (ArgoCD sync on merge)
- ⬜ YAML values updated programmatically in pipeline

### Section 3: Platform Engineering — Meet Backstage

**Backstage Setup**
- ⬜ Backstage deployed via Docker
- ⬜ GitHub OAuth configured
- ⬜ Plugins downloaded and added to frontend
- ⬜ Backstage authentication working

### Section 4: Backstage Software Catalog

- ⬜ Group entities configured
- ⬜ `catalog-info.yaml` (service registered)
- ⬜ Existing components registered in catalog

### Section 5: Backstage TechDocs

- ⬜ `docs/` folder with Markdown documentation
- ⬜ `mkdocs.yml` configured
- ⬜ TechDocs plugin installed and configured
- ⬜ Docs rendering in Backstage

### Section 6: Backstage Software Templates

**Setup**
- ⬜ Backstage Actions plugin installed and configured
- ⬜ GitHub integrated with Backstage
- ⬜ Dedicated GitHub repo for Software Templates

**Template Development**
- ⬜ Input parameters defined
- ⬜ Template steps defined
- ⬜ `catalog-info.yaml` wired into template
- ⬜ DevOps project rewritten as a Backstage Software Template
- ⬜ Template renders successfully in Backstage

**GitHub Orgs & Distributed Builds**
- ⬜ GitHub organization configured
- ⬜ Distributed builds on GitHub Orgs
- ⬜ Repository properties configurable from Backstage Actions

**ArgoCD Integration**
- ⬜ ArgoCD operations automated via Backstage Templates
- ⬜ Full template tested end-to-end (repo create → ArgoCD sync)

**CI/CD View**
- ⬜ CI/CD view customized in Backstage

### Section 7: Backstage in Production Mode

- ⬜ PostgreSQL database provisioned for Backstage
- ⬜ Production config file (`app-config.production.yaml`)
- ⬜ Backstage backend built as a bundle
- ⬜ Production-ready Dockerfile for Backstage
- ⬜ Backstage + Postgres deployed as Docker containers
### Section 8: Backstage on Kubernetes

- ⬜ PostgreSQL deployed to K8s via Helm
- ⬜ Kubernetes manifests for Backstage
- ⬜ Backstage Docker image pushed to registry
- ⬜ Backstage deployed and running on Kubernetes

### Section 9: You Did It! ⬜
Course complete.

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
