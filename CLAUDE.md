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
