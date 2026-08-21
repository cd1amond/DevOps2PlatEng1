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

## Architecture

The liveness/readiness probes in `values.yaml` currently point to `/` (root), which returns a 404 since the app only handles `/api/v1/*`. Update probes to `/api/v1/healthz` when deploying to a real cluster.

## Local Environment Guardrails

- **Two kind clusters share one Rancher Desktop VM (~9.7Gi RAM), only one fits at a time.** `forge-idp` (context `forge-idp`, container `kind-control-plane`) is the user's main dev cluster — never delete it or its namespaces. `devops-course` (context `kind-devops-course`, container `devops-course-control-plane`) is this course's cluster. Switch via:
  ```bash
  docker stop kind-control-plane && docker start devops-course-control-plane && kubectl config use-context kind-devops-course
  docker stop devops-course-control-plane && docker start kind-control-plane && kubectl config use-context forge-idp
  ```
- **Corporate network blocks direct image pulls from inside kind nodes** (TLS inspection breaks cert validation) — see the `fix-kind-registry-tls` skill for the Harbor pull-through-proxy mapping and the CA-cert fix runbook. `forge-idp`'s node has NOT had the TLS fix applied yet — same issue likely exists there.
- No memory MCP server in this environment (removed — was unreliable). Durable notes belong here in CLAUDE.md, not in a memory tool.
- **Course uses `<service>.test.com` as placeholder Ingress hostnames** (confirmed against instructor's reference repo, `github.com/ricardoandre97/python-app`, which hardcodes `python-app.test.com` in `k8s/ingress.yaml`). Not real DNS — resolved locally via `/etc/hosts` (`127.0.0.1 python-app.test.com argocd.test.com`, already added). Expect more of these (`argocd.test.com`, later probably `backstage.test.com`).
- **NodePort services from `devops-course` are NOT reachable from the Mac browser.** `devops-course` was created via plain `kind create cluster --name devops-course`, with no `extraPortMappings`, so only the Kubernetes API port is published to the host (`docker port devops-course-control-plane`). ingress-nginx's NodePort (80→something like 31254) only resolves inside the Rancher Desktop VM (`rdctl shell -- curl ...` works; `curl` from the Mac times out, even hitting the node's bridge IP directly). Ingress resources apply and route correctly — verified via `rdctl shell`— this is purely a host↔VM port-publishing gap, not a bug in the Ingress config. Fix requires deleting and recreating the cluster with `extraPortMappings` for 80/443, then redeploying everything (ArgoCD, python-app, cert-manager, ingress-nginx, plus reapplying the CA-cert fix above). Not done as of 2026-08-10 — revisit if a later lecture needs real browser access to `*.test.com`.
