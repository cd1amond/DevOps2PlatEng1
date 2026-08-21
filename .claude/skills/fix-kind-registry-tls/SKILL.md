---
name: fix-kind-registry-tls
description: Use when a kind node in this project's local cluster can't pull an image from quay.io/ghcr.io/registry.k8s.io due to a TLS/cert validation error — fixes the corporate MITM proxy cert not propagating into the kind node's container filesystem, and gives the Harbor pull-through-proxy path as an alternative.
---

# Fix kind node registry TLS pull failures

## Option 1: route through the Harbor pull-through proxy

`harbor.lsdeint.leidos.com/<cache>/<upstream-path>:<tag>` — caches are `quay-proxy-cache`, `docker-hub-proxy-cache` (official images need `/library/` prefix, e.g. `docker-hub-proxy-cache/library/redis`), `ghcr-proxy-cache`, `gcr-proxy-cache`, `dhi-proxy-cache`. No `registry.k8s.io` mirror exists in Harbor (confirmed via `GET /api/v2.0/projects` — only those 5 registry-backed caches exist). `cdiamond/python-app` (this project's own image) is public and doesn't need rewriting.

## Option 2: fix the root cause (recommended — pulls work directly afterward)

Rancher Desktop's VM trusts the corporate MITM root `Leidos Cloud PKI Root CA-2` (system trust store), but that cert never propagates into a kind node's separate container filesystem — so containerd inside the node can't validate TLS to any registry the corporate proxy intercepts, even though `docker pull` on the host works fine.

Fix per kind node (not persisted — redo after `kind delete cluster` + recreate):

```bash
# extract the root CA from the VM (find the file first: loop rd-N.crt subjects for "Leidos Cloud PKI Root CA-2")
rdctl shell -- sudo cat /usr/local/share/ca-certificates/rd-12.crt > /tmp/leidos-root-ca2.crt   # filename may differ
docker cp /tmp/leidos-root-ca2.crt <node-container>:/usr/local/share/ca-certificates/leidos-root-ca2.crt
docker exec <node-container> update-ca-certificates --fresh   # plain update-ca-certificates is not enough, must be --fresh
docker exec <node-container> systemctl restart containerd
# verify: docker exec <node-container> sh -c 'echo | openssl s_client -connect registry.k8s.io:443 -CAfile /etc/ssl/certs/ca-certificates.crt 2>&1 | grep "Verify return code"'  → should say "(ok)"
```

Once fixed, images pull directly from `registry.k8s.io`/quay.io/ghcr.io without needing the Harbor rewrite. Applied to `devops-course-control-plane` on 2026-08-10 to unblock ingress-nginx (no Harbor mirror exists for `registry.k8s.io`).
