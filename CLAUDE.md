# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Course Context

Udemy course: *From DevOps to Platform Engineering: Master Backstage & IDPs*. This repo is the hands-on project workspace covering:
- Docker + Kubernetes + ArgoCD deployments
- GitHub Actions CI/CD pipelines
- Backstage Internal Developer Portal (IDP) setup
- TechDocs (Documentation as Code)
- Backstage Software Templates

## What This Is

A minimal Flask API containerized with Docker and deployed to Kubernetes. The app exposes two endpoints:
- `GET /api/v1/details` — returns current time and hostname (used to demonstrate pod identity in K8s)
- `GET /api/v1/healthz` — health check, returns `{"status": "up"}`

## Running Locally

```bash
pip install -r requirements.txt
python src/app.py          # runs on 0.0.0.0:5000
```

## Docker

```bash
docker build -t cdiamond/python-app:v2 .
docker run -p 5000:5000 cdiamond/python-app:v2
```

## Kubernetes

Raw manifests are in `k8s/` (currently empty stubs). The Helm chart under `charts/python-app/` is the primary deployment path:

```bash
# Lint and dry-run
helm lint charts/python-app
helm template python-app charts/python-app

# Install / upgrade
helm upgrade --install python-app charts/python-app

# With custom values
helm upgrade --install python-app charts/python-app --set image.tag=v3
```

## Architecture

```
src/app.py          Flask app (single file, no blueprints)
Dockerfile          python:3.10-alpine, copies src/, runs app.py directly
k8s/                Raw YAML manifests (deploy, service, ingress) — stubs
charts/python-app/  Helm chart wrapping the same deployment
  values.yaml       Image: cdiamond/python-app:v2, port 5000, ClusterIP service
  templates/        Deployment, Service, Ingress, HTTPRoute (gateway-api), helpers
```

The Helm chart supports both classic Ingress (`ingress.enabled`) and Gateway API HTTPRoute (`httpRoute.enabled`). Both are disabled by default. The `containerPort` and `service.port` both use the value from `values.yaml` (`5000`), so changing the port only requires updating `values.yaml`.

The liveness/readiness probes in `values.yaml` currently point to `/` (root), which returns a 404 since the app only handles `/api/v1/*`. Update probes to `/api/v1/healthz` when deploying to a real cluster.

## Local Environment Guardrails

- **Two kind clusters share one Rancher Desktop VM (~9.7Gi RAM), only one fits at a time.** `forge-idp` (context `forge-idp`, container `kind-control-plane`) is the user's main dev cluster — never delete it or its namespaces. `devops-course` (context `kind-devops-course`, container `devops-course-control-plane`) is this course's cluster. Switch via:
  ```bash
  docker stop kind-control-plane && docker start devops-course-control-plane && kubectl config use-context kind-devops-course
  docker stop devops-course-control-plane && docker start kind-control-plane && kubectl config use-context forge-idp
  ```
- **Corporate network blocks direct pulls** from quay.io/ghcr.io/registry.k8s.io (TLS inspection breaks cert validation) — but only from inside kind nodes, not from the host. Route images through the Harbor pull-through proxy when possible: `harbor.lsdeint.leidos.com/<cache>/<upstream-path>:<tag>` — caches are `quay-proxy-cache`, `docker-hub-proxy-cache` (official images need `/library/` prefix, e.g. `docker-hub-proxy-cache/library/redis`), `ghcr-proxy-cache`, `gcr-proxy-cache`, `dhi-proxy-cache`. No `registry.k8s.io` mirror exists in Harbor (confirmed via `GET /api/v2.0/projects` — only those 5 registry-backed caches exist). `cdiamond/python-app` (this project's own image) is public and doesn't need rewriting.
- **Root cause of the TLS block, and the real fix (found 2026-08-10):** Rancher Desktop's VM trusts the corporate MITM root `Leidos Cloud PKI Root CA-2` (system trust store), but that cert never propagates into a kind node's separate container filesystem — so containerd inside the node can't validate TLS to any registry the corporate proxy intercepts, even though `docker pull` on the host works fine. Fix per kind node (not persisted — redo after `kind delete cluster` + recreate):
  ```bash
  # extract the root CA from the VM (find the file first: loop rd-N.crt subjects for "Leidos Cloud PKI Root CA-2")
  rdctl shell -- sudo cat /usr/local/share/ca-certificates/rd-12.crt > /tmp/leidos-root-ca2.crt   # filename may differ
  docker cp /tmp/leidos-root-ca2.crt <node-container>:/usr/local/share/ca-certificates/leidos-root-ca2.crt
  docker exec <node-container> update-ca-certificates --fresh   # plain update-ca-certificates is not enough, must be --fresh
  docker exec <node-container> systemctl restart containerd
  # verify: docker exec <node-container> sh -c 'echo | openssl s_client -connect registry.k8s.io:443 -CAfile /etc/ssl/certs/ca-certificates.crt 2>&1 | grep "Verify return code"'  → should say "(ok)"
  ```
  Once fixed, images pull directly from `registry.k8s.io`/quay.io/ghcr.io without needing the Harbor rewrite. This was applied to `devops-course-control-plane` to unblock ingress-nginx (no Harbor mirror exists for `registry.k8s.io`). **`forge-idp`'s node (`kind-control-plane`) has NOT had this fix applied yet** — same issue likely exists there, apply the same steps if it hits registry TLS errors.
- No memory MCP server in this environment (removed — was unreliable). Durable notes belong here in CLAUDE.md, not in a memory tool.
