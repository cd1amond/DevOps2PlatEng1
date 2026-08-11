# RESTART KIT — DevOps2PlatEng1: Course Progress (Section 2 complete, starting Section 3)

**Created:** 2026-08-10, end of Section 2 (lecture 60 + quiz 7 complete, next up: Section 3 Backstage)
**For:** a fresh session to continue hands-on course work alongside the user.
**Status:** Section 2 (DevOps skills: Docker/K8s/Helm/ArgoCD/CI/CD) fully complete and verified working end-to-end. Repo pushed to GitHub @ c4642ff. No implementation plan — this is a learning/lab session, not a code-generation task.
**Approval gate:** user pasting the kickoff prompt below is the go — do not re-plan or re-ask.

---

## TL;DR

This is a Udemy course project repo (`cd1amond/DevOps2PlatEng1`). The user is working through the course interactively — you help them debug, write config/code, tick off README progress, commit, and push. You are a co-pilot, not an autonomous executor. Wait for the user to tell you what lecture they're on and what they're doing.

Since the previous restart kit (2026-08-04, lecture 48), the user completed all of Section 2: self-hosted GitHub Actions runner, ArgoCD Application + sync, and a fully working GitOps loop (push → build → tag-bump → sync → new pod live). Several genuine gaps in the course material were found and fixed along the way — see "Verified ground truth" below. The user is now at lecture 60 / quiz 7, about to start Section 3 (Backstage).

**Do not re-plan, do not re-derive. Do not re-litigate decisions already made below.**

---

## Verified ground truth (trust these)

### Cluster / environment
- **Two kind clusters share one Rancher Desktop VM (~9.7Gi RAM)** — only one fits comfortably at a time. `forge-idp` (context `forge-idp`, container `kind-control-plane`) is the user's main dev cluster — **never delete it or its namespaces**. `devops-course` (context `kind-devops-course`, container `devops-course-control-plane`) is this course's cluster. Switch via:
  ```bash
  docker stop kind-control-plane && docker start devops-course-control-plane && kubectl config use-context kind-devops-course
  docker stop devops-course-control-plane && docker start kind-control-plane && kubectl config use-context forge-idp
  ```
- **The Rancher Desktop VM's outbound networking drops intermittently** (observed twice this session, hours apart, no clear trigger). Symptom: `context deadline exceeded` / connection timeouts on anything the VM tries to reach externally (GitHub, registries) — DNS resolves fine, TCP just hangs. Node-level (`docker exec <node> ...`) and VM-level (`rdctl shell -- ...`) checks both fail identically when this happens. **Fix: quit and relaunch Rancher Desktop entirely** (`osascript -e 'quit app "Rancher Desktop"'` then `open -a "Rancher Desktop"`, wait ~25s for `docker info` to respond). This is NOT a kind/K8s config issue — don't waste time debugging DNS or containerd config when this happens, just restart the app.
- **As of this kit's creation, the VM's egress is down again** (confirmed: `rdctl shell -- curl https://github.com` times out). ArgoCD's `python-app` Application shows `SYNC STATUS: Unknown` with a `ComparisonError` (`context deadline exceeded` reaching GitHub) as a direct result. **First thing to do in a fresh session: restart Rancher Desktop**, then verify with `rdctl shell -- curl -sS -m 5 -o /dev/null -w "%{http_code}\n" https://github.com` (expect `200`-ish, not a timeout).

### Corporate registry / TLS (the big one — cost hours to find)
- **Corporate network TLS-inspects/blocks direct pulls** from quay.io/ghcr.io/registry.k8s.io — but the block is INSIDE kind nodes' containerd only, not the Mac host and not the Rancher VM itself (`docker pull` on the Mac host works fine for the same images).
- **Root cause (found 2026-08-10):** Rancher Desktop's VM trusts the corporate MITM root `Leidos Cloud PKI Root CA-2` in its system trust store, but that cert never propagates into a kind node's separate container filesystem. Fixed on `devops-course-control-plane` by extracting the cert from the VM and injecting it into the node's containerd trust store — full steps are in `CLAUDE.md` under "Local Environment Guardrails". **This fix is NOT persisted** — redo it if the cluster is ever recreated (`kind delete cluster` + `kind create cluster`).
- **`forge-idp`'s node (`kind-control-plane`) has NOT had this fix applied.** If forge-idp work hits a registry TLS error, apply the same steps (documented in this repo's CLAUDE.md, generic enough to reuse there).
- **Harbor pull-through proxy** (`harbor.lsdeint.leidos.com/<cache>/<upstream-path>:<tag>`) is the general-purpose workaround when the CA fix isn't available or convenient — caches: `quay-proxy-cache`, `docker-hub-proxy-cache` (needs `/library/` prefix for official images), `ghcr-proxy-cache`, `gcr-proxy-cache`, `dhi-proxy-cache`. **No `registry.k8s.io` mirror exists in Harbor** (confirmed via Harbor's own `GET /api/v2.0/projects` — only those 5 registry-backed caches exist). This is why ingress-nginx specifically needed the CA-fix approach rather than Harbor.

### ArgoCD
- Installed via `kubectl apply` (NOT Helm) in namespace `argocd`, version v2.13.6, all 7 pods healthy under normal conditions.
- **Repo connection needs explicit credentials** — the repo (`cd1amond/DevOps2PlatEng1`) is private. Registering it in ArgoCD's UI with just the bare URL fails with `authentication required: Repository not found` (GitHub returns 404-not-403 for private repos to unauthenticated requests). Fixed via a `kubectl create secret generic` with `type=git`, `url`, `username=cd1amond`, `password=<PAT>`, labeled `argocd.argoproj.io/secret-type=repository`.
- **argocd CLI version MUST match the server version (v2.13.6).** Installing "latest" (resolved to v3.5.0 during this session) causes `argocd login` to hang and fail with `gRPC connection not ready: context deadline exceeded` — NOT a clear version-mismatch error, just a silent timeout. The CD workflow step now pins `https://github.com/argoproj/argo-cd/releases/download/v2.13.6/argocd-linux-arm64` (arm64 because the runner pod is aarch64 — Apple Silicon under Rancher Desktop).
- ArgoCD admin password: `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d` (rotates if the secret is regenerated — always re-fetch, don't hardcode).
- ArgoCD UI access: `kubectl port-forward svc/argocd-server -n argocd 8080:443` then browse `https://localhost:8080` (self-signed cert, click through the warning).

### GitHub Actions self-hosted runner
- Installed via `actions-runner-controller` Helm chart, namespace `actions-runner-system`. `RunnerDeployment` named `self-hosted-runners`, targets repo `cd1amond/DevOps2PlatEng1`.
- **Runner pods are ephemeral** — a fresh pod spins up per job, so anything installed mid-job (like the argocd CLI) does NOT persist to the next run. The workflow re-installs argocd CLI every CD job run; this is intentional/necessary, not a bug to fix.
- **The GitHub PAT used for the runner's `authSecret` (`ghp_o2FuDqkm...`) was exposed in plaintext** in this session's terminal/transcript. **User should rotate it** — if rotated, both the runner's `authSecret` (`helm upgrade` with the new token) and the ArgoCD repo credential secret (`kubectl create secret` again, or `kubectl edit secret repo-devops2plateng1 -n argocd`) need updating to match.

### CI/CD pipeline (`.github/workflows/cicd.yaml`) — fully working GitOps loop
- Triggers on `push` to `src/**` on `main`, or `workflow_dispatch`.
- `ci` job: checks out repo, builds+pushes `cdiamond/python-app:<6-char-commit-sha>` to Docker Hub, then uses `yq` (preinstalled on `ubuntu-latest`, do NOT `snap install yq` — that's a different, incompatible tool) to bump `charts/python-app/values.yaml`'s `image.tag`, commits, and pushes back to `main`. Needs `permissions: contents: write` on the job.
- `cd` job (`runs-on: self-hosted`): installs argocd CLI v2.13.6, logs in via `ARGOCD_USERNAME`/`ARGOCD_PASSWORD` secrets, runs `argocd app sync python-app`.
- **Verified end-to-end multiple times**: a push to `src/app.py` correctly flows through build → tag bump → git commit → ArgoCD sync → new pod rolls in (old one terminates), all automatically.
- Endpoint renamed: `/api/v1/details` → `/api/v1/info` (adds a `deployed_on: "kubernetes"` field). `/api/v1/healthz` unchanged.

### Ingress
- `ingress-nginx` v1.11.3 deployed via the kind-flavored upstream manifest (`deploy/static/provider/kind/deploy.yaml`), requires the node to be labeled `ingress-ready=true` to schedule (`kubectl label node devops-course-control-plane ingress-ready=true`).
- `k8s/ingress.yaml` defines a `python-app` Ingress on host `python-app.test.com`, `ingressClassName: nginx` (must be explicit — no default IngressClass is set in this cluster).
- **`*.test.com` hostnames are a course convention, not real DNS** — confirmed by checking the instructor's reference repo (`github.com/ricardoandre97/python-app`), which hardcodes `python-app.test.com` the same way. Resolved locally via `/etc/hosts` (`127.0.0.1 python-app.test.com argocd.test.com` — already added on this Mac).
- **The Ingress routes correctly but is NOT reachable from the Mac browser.** `devops-course` was created via plain `kind create cluster --name devops-course` with no `extraPortMappings`, so only the Kubernetes API port is published to the host — the ingress-nginx NodePort only resolves from inside the Rancher Desktop VM (`rdctl shell -- curl ...` works; `curl` from the Mac, even at the node's bridge IP, times out). This is a known, accepted gap — fixing it requires recreating the cluster with port mappings and redeploying everything. Not planned unless a future lecture needs real browser access.

### cert-manager
- v1.8.2 deployed via the upstream release manifest, all 3 pods healthy. No further course usage yet as of this kit.

### GitHub / git
- **Repo is private**: `github.com/cd1amond/DevOps2PlatEng1`.
- **`gh` CLI needed the `workflow` OAuth scope added** mid-session (`gh auth refresh -h github.com -s workflow`, interactive device-flow — needs the user to open a browser) after a push was rejected for touching `.github/workflows/*` without that scope.
- GitHub secrets configured: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `ARGOCD_USERNAME`, `ARGOCD_PASSWORD`.
- `WebSearch` tool 400s on this backend — use `ask-codex` for web searches, or `WebFetch` on a known URL.

---

## What is DONE / NOT done

### Done (Section 2 fully complete, committed, pushed, @ c4642ff)
- Flask API, Dockerfile, Helm chart, raw K8s manifests (deploy/service/ingress)
- Local kind cluster (`devops-course`) with ArgoCD, ingress-nginx, cert-manager, actions-runner-controller all deployed and healthy
- Full CI/CD GitOps loop: build → push → tag-bump → git commit → ArgoCD sync → live pod update, verified working multiple times
- `python-app` Ingress resource (routing verified, browser access from Mac not yet possible — see ground truth)
- Corporate CA trust fix applied to `devops-course`'s node (documented, not persisted)

### Not done (starting at Section 3)
- All of Backstage (Sections 3–9): Backstage deployment, GitHub OAuth, Software Catalog, TechDocs, Software Templates, production mode, Kubernetes deployment of Backstage itself
- Browser access to `*.test.com` hostnames (would need cluster recreation with `extraPortMappings` — deferred)
- Rotating the exposed GitHub PAT (user's call, not yet done as of this kit)
- Applying the corporate CA cert fix to `forge-idp`'s node (only needed if/when that cluster hits a registry TLS error)

---

## Preflight (run at start of fresh session)

```bash
# 0. If Rancher Desktop's VM networking is down (check first — see ground truth above):
osascript -e 'quit app "Rancher Desktop"'
# wait for it to fully quit, then:
open -a "Rancher Desktop"
# wait ~25s, then verify:
rdctl shell -- curl -sS -m 5 -o /dev/null -w "%{http_code}\n" https://github.com   # expect 200-ish, not a timeout

# 1. Verify cluster is up (both containers may restart on their own after a Rancher restart —
#    stop forge-idp's if it comes up, keep only devops-course running per the shared-VM guardrail)
docker ps -a --filter "name=kind-control-plane" --filter "name=devops-course-control-plane" --format '{{.Names}}\t{{.Status}}'
docker stop kind-control-plane 2>/dev/null   # only if it's running — forge-idp stays stopped during coursework
kubectl config use-context kind-devops-course
kubectl get nodes
kubectl get pods -n argocd
kubectl get pods -n python
kubectl get application python-app -n argocd   # expect Synced/Healthy once egress is back

# 2. Verify git state
git -C /Users/diamondcj/dev/DevOps2PlatEng1 status
git -C /Users/diamondcj/dev/DevOps2PlatEng1 pull   # pick up any bot auto-commits from the pipeline
git -C /Users/diamondcj/dev/DevOps2PlatEng1 log --oneline -5
```

---

## Kickoff instructions (fresh session executes these exactly)

1. Read this file fully.
2. Run the Preflight block above — fix Rancher Desktop networking FIRST if needed, before touching anything else.
3. Read `README.md` to see current progress state.
4. Read `CLAUDE.md` for codebase context (includes the full CA-cert fix steps and other environment guardrails).
5. Tell the user: "Ready. I have full context from the restart kit — Section 2 is done, cluster's healthy [or: I just fixed Rancher Desktop's networking]. What lecture are you on?"
6. Wait for the user — this is an interactive learning session. Help them as they work through each lecture. When they complete something, update the README checklist and commit.

**Do not autonomously start implementing anything. The user drives the pace. Section 3 is Backstage — expect a much heavier stack (Node.js, Docker Compose or K8s, GitHub OAuth app registration, Postgres later) than Section 2's single Flask app.**

---

## Sharp edges the executor will hit

- **Git pushes:** `gh` CLI now has `workflow` scope (fixed this session). If a push to `.github/workflows/*` is rejected again for missing scope, run `gh auth refresh -h github.com -s workflow` (interactive, needs the user to complete a device-flow in a browser).
- **Any new K8s images pulled directly (not via Harbor)** will now work on `devops-course`'s node thanks to the CA fix — but if the cluster is ever recreated, redo the CA fix first (steps in CLAUDE.md) before assuming direct registry pulls will work.
- **Runner pods are ephemeral** — don't be surprised if a tool "disappears" between workflow runs; that's by design, not a regression.
- **ArgoCD CLI version drift**: if Backstage sections introduce a different ArgoCD interaction (e.g., a newer ArgoCD instance, or `argocd` CLI used outside the pinned workflow step), re-check version compatibility — don't assume "latest" is safe.
- **`sudo` writes to `/etc/hosts`** need the user to run them directly (permission-denied for the assistant) — if more `*.test.com` entries are needed for Backstage, ask the user to run the `sudo sh -c 'echo ... >> /etc/hosts'` command themselves.
- **Course material has real gaps** (confirmed multiple times this session: missing ArgoCD Application manifest instructions, missing argocd CLI on the runner image, a "latest" version pin that silently breaks, an ingress domain example that assumes port-mapping never set up, a PAT introduced but never wired in). Expect more of these in the Backstage sections — verify course steps against actual repo/cluster state rather than assuming the walkthrough is complete.
