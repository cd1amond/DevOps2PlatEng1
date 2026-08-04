# RESTART KIT — DevOps2PlatEng1: Course Progress

**Created:** 2026-08-04, end of Section 2 CD setup (lecture 47 complete, starting lecture 48)
**For:** a fresh session to continue hands-on course work alongside the user.
**Status:** repo live on GitHub @ 9bc7972. No implementation plan — this is a learning/lab session, not a code-generation task.
**Approval gate:** user pasting the kickoff prompt below is the go — do not re-plan or re-ask.

---

## TL;DR

This is a Udemy course project repo (`cd1amond/DevOps2PlatEng1`). The user is working through the course interactively — you help them debug, write config/code, tick off README progress, commit, and push. You are a co-pilot, not an autonomous executor. Wait for the user to tell you what lecture they're on and what they're doing.

**Do not re-plan, do not re-derive.**

---

## Verified ground truth (trust these)

- **Cluster:** kind cluster named `kind-control-plane`, running locally via Rancher Desktop. Namespace `python` has the Flask app deployed.
- **ArgoCD:** installed via `kubectl apply` (NOT Helm) in namespace `argocd`, version v2.13.6. All 7 pods running. Images pulled through Harbor proxy cache (`harbor.lsdeint.leidos.com`). Initial admin password was retrieved this session — check `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d` to get the current value.
- **Harbor proxy caches available:** `quay-proxy-cache` (quay.io), `docker-hub-proxy-cache` (Docker Hub), `ghcr-proxy-cache` (ghcr.io), `gcr-proxy-cache` (gcr.io). No `registry.k8s.io` mirror — ingress-nginx cannot be installed.
- **Corporate network constraint:** `registry.k8s.io` is blocked (TLS x509 cert error from Leidos SSL inspection). All K8s images must come through Harbor proxy caches. `WebSearch` tool 400s on this backend — use `ask-codex` for web searches.
- **Ingress:** NOT configured (ingress-nginx namespace was deleted). Course ingress exercises were skipped due to missing `registry.k8s.io` mirror. Port-forward is the workaround.
- **Docker Hub:** repo is `cdiamond/python-app` (NOT `diamondcj/python-app`). CI workflow fixed to use correct username.
- **GitHub PAT:** needs `repo` + `workflow` + `write:packages` scopes. macOS keychain caches old tokens — workaround is `printf ... | git credential-osxkeychain store` or embedding token in remote URL temporarily. Token exposed in chat this session — user should rotate.
- **CI workflow:** `.github/workflows/ci.yaml` — triggers on `src/**` push to main OR `workflow_dispatch`. Builds and pushes `cdiamond/python-app:<6-char-sha>` to Docker Hub. Verified working (run 30867442969 succeeded).
- **Helm probes fix:** `charts/python-app/values.yaml` liveness/readiness probes updated to `/api/v1/healthz` (was `/`, caused CrashLoopBackOff).
- **Starship:** `~/.config/starship.toml` created with `command_timeout = 2000` at top level (not inside `[python]`).
- **Instructor's repo:** `https://github.com/ricardoandre97/python-app/` — useful reference for expected file structure.

---

## What is DONE / NOT done

### Done (committed, pushed, @ 9bc7972)
- Flask API, Dockerfile, Helm chart, raw K8s manifest stubs
- `.gitignore` (macOS, Python, Helm, secrets)
- `README.md` with full course artifact checklist
- `CLAUDE.md` with codebase guide
- Helm chart deployed to local kind cluster (namespace `python`)
- ArgoCD deployed (all pods running)
- CI GitHub Actions workflow (build + push with dynamic tags)
- GitHub secrets: `DOCKERHUB_USERNAME=cdiamond`, `DOCKERHUB_TOKEN`

### Not done (starting at lecture 48)
- Self-hosted GitHub Actions runner
- CD workflow (ArgoCD sync)
- YAML values updated programmatically in pipeline
- ArgoCD Application manifest + app sync
- All of Sections 3–9 (Backstage)

---

## Preflight (run at start of fresh session)

```bash
# Verify cluster is up
kubectl get nodes
kubectl get pods -n python
kubectl get pods -n argocd

# Verify git state
git -C /Users/diamondcj/dev/DevOps2PlatEng1 status
git -C /Users/diamondcj/dev/DevOps2PlatEng1 log --oneline -5
```

---

## Kickoff instructions (fresh session executes these exactly)

1. Read this file fully.
2. Read `README.md` to see current progress state.
3. Read `CLAUDE.md` for codebase context.
4. Tell the user: "Ready. I have full context from the restart kit. What lecture are you on?"
5. Wait for the user — this is an interactive learning session. Help them as they work through each lecture. When they complete something, update the README checklist and commit.

**Do not autonomously start implementing anything. The user drives the pace.**

---

## Sharp edges the executor will hit

- **Git pushes:** keychain is correctly configured with current PAT (fixed end of session 1). Plain `git push` works. If it breaks again, fix with `printf 'protocol=https\nhost=github.com\nusername=cd1amond\npassword=<PAT>\n' | git credential-osxkeychain store`.
- **Any new K8s images** must be pulled through Harbor proxy caches. Check which proxy cache covers the image's registry before deploying.
- **CD workflow (lecture 48–60)** uses a self-hosted runner — the runner registers to GitHub and runs on the user's Mac. It needs `kubectl`, `argocd` CLI, and `python`/`pip` (for `yq`) in PATH.
- **ArgoCD was installed via kubectl apply, not Helm** — don't try to manage it with Helm.
- **Course mixes kubectl and Helm** inconsistently — this is a known course issue. When in doubt, match what's already deployed (kubectl for ArgoCD, Helm for python-app).
