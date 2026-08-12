# RESTART KIT — DevOps2PlatEng1: Course Progress (Section 3 in progress, lecture ~68, OAuth mid-setup)

**Created:** 2026-08-12, mid-Section-3 (Backstage deployed via Docker and reachable; GitHub OAuth wiring in progress, not yet tested end-to-end).
**For:** a fresh session to continue hands-on course work alongside the user.
**Status:** Section 2 fully complete (see prior kit, `docs/plans/2026-08-10-devops2plateng-lecture60-RESUME.md`, superseded by this one for current state). Section 3: Backstage app scaffolded and serving locally; GitHub OAuth App config just fixed (had two bugs, corrected below) but not yet confirmed working. No implementation plan — this is a learning/lab session, not a code-generation task.
**Approval gate:** user pasting the kickoff prompt below is the go — do not re-plan or re-ask.

---

## TL;DR

This is a Udemy course project repo (`cd1amond/DevOps2PlatEng1`). The user is working through the course interactively — you help them debug, write config/code, tick off README progress, commit, and push. You are a co-pilot, not an autonomous executor. Wait for the user to tell you what lecture they're on and what they're doing.

Since the last restart kit (lecture 60), the user got through roughly lecture 61–68 (Section 3, Backstage). Backstage itself is **not** part of this git repo — it's a separate scaffold at `~/dev/backstage-app/backstage/`, built with `@backstage/create-app`, run inside a `node:22-bookworm-slim` Docker container. Getting it running required diagnosing three separate environment bugs this session (all fixed, details below). GitHub OAuth is the current task — client ID/secret wiring is in progress, config bugs just fixed, sign-in not yet tested.

**Do not re-plan, do not re-derive. Do not re-litigate decisions already made below.**

---

## Verified ground truth (trust these)

### Cluster / environment (carried over + updated from lecture-60 kit)
- **Two kind clusters share one Rancher Desktop VM (~9.7Gi RAM)** — `forge-idp` (context `forge-idp`, container `kind-control-plane`) never delete. `devops-course` (context `kind-devops-course`, container `devops-course-control-plane`) is this course's cluster. Same stop/start dance as before to switch.
- **NEW — a third, undocumented workload set is always running on this VM**: `kubectl config get-contexts` shows a `rancher-desktop` context (cluster `rancher-desktop`, node `lima-rancher-desktop`, k3s v1.32.11) — this is Rancher Desktop's own built-in Kubernetes, separate from both kind clusters, running continuously for 22+ days. It hosts `argocd`, `cnpg-system` (Postgres), `external-secrets`, `flux-system`, `keycloak`, `kgateway-system` — a real, presumably-in-use stack, not course-related. **Do not touch/disable it without asking the user first** — it wasn't mentioned in any prior kit and may be load-bearing for something outside this course. It alone uses ~5.4–6G of the VM's 9.7G RAM at idle, meaning any kind cluster + this course's Docker containers are competing for the remaining ~3–4G. This directly caused symptoms below (native module build flakiness) — check `rdctl shell -- free -h` if anything memory-sensitive misbehaves.
- **VM egress still drops intermittently** (same as lecture-60 kit — confirmed again this session, hours into the session, `rdctl shell -- curl https://github.com` timed out while ArgoCD's `python-app` app showed `SYNC STATUS: Unknown` / `ComparisonError`). Fix is still: quit + relaunch Rancher Desktop entirely, wait ~25–30s, verify with the curl check, then `kubectl patch application python-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'` to force ArgoCD to re-check instead of waiting on its own poll interval.
- **First thing to check in a fresh session**: `rdctl shell -- curl -sS -m 5 -o /dev/null -w "%{http_code}\n" https://github.com` — if it times out, restart Rancher Desktop before anything else (registry pulls, git operations, and Backstage's own OAuth/plugin downloads all depend on this).

### Corporate CA trust — now confirmed to affect Node/npm too, not just containerd
- Prior kits documented the corporate MITM root (`Leidos Cloud PKI Root CA-2`) breaking kind-node **registry** pulls. This session found the **same root cause breaks `node-gyp` native module builds** inside a plain `docker run` dev container (not a kind node) — `node-gyp` tries to fetch Node headers from `nodejs.org` and gets `self-signed certificate in certificate chain`.
- **Fix for any future Docker container that needs outbound HTTPS on this network** (not just kind nodes):
  1. Extract the cert from the VM: find the right file first (`rdctl shell -- sudo sh -c 'for f in /usr/local/share/ca-certificates/rd-*.crt; do openssl x509 -in "$f" -noout -subject 2>&1; done' | grep -i leidos` — it's `rd-12.crt`, subject `O=Leidos, CN=Leidos Cloud PKI Root CA-2`), then `rdctl shell -- sudo cat /usr/local/share/ca-certificates/rd-12.crt > /tmp/leidos-root-ca2.crt`.
  2. `docker cp /tmp/leidos-root-ca2.crt <container>:/usr/local/share/ca-certificates/leidos-root-ca2.crt` then inside the container: `apt-get install -y ca-certificates && update-ca-certificates --fresh`.
  3. **Node does NOT read the system trust store by default** — you additionally need `export NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/leidos-root-ca2.crt` in the shell running `npm`/`npx`/`yarn`, or native module builds still fail with the same TLS error even after step 2.
  4. This is not persisted — redo per container if it's recreated.

### Backstage / Node version (new this session — course material is stale here)
- **Backstage now requires Node 22 or 24** (confirmed via live web search, Backstage v1.46.0 release notes) — the course's `node:18-bookworm-slim` instruction is out of date (Node 18 has been EOL since April 2025). Use `node:22-bookworm-slim`.
- **The `-slim` Node image has no compiler toolchain.** `@backstage/create-app` needs to build several native modules (`better-sqlite3`, `isolated-vm`, `tree-sitter`, `cpu-features`, etc.). Install `python3 make g++` before running `npx @backstage/create-app@latest`, in addition to the CA-cert fix above (both are needed — toolchain fixes "command not found"-class failures, the CA cert fixes the TLS failures node-gyp hits mid-build).
- **The frontend dev server binds to loopback only by default — Docker's `-p 3000:3000` cannot reach it.** Confirmed via `docker exec <container> cat /proc/net/tcp` / `tcp6`: the frontend listener was on `::1`/`127.0.0.1`, not `0.0.0.0`/`::`, even with `NODE_OPTIONS=--dns-result-order=ipv4first` set (that only changed which loopback family it used, not the fact that it was loopback-only). The actual fix (confirmed via live web search against current Backstage docs): add to `app-config.yaml`:
  ```yaml
  app:
    listen:
      host: 0.0.0.0
  ```
  There is no `--host` flag on `backstage-cli repo start` — this must be set in config. The backend (`backend.listen.host`) already had a commented-out example for this in the scaffolded config; the frontend equivalent (`app.listen.host`) is undocumented in the generated file's comments but works.
- **`yarn dev` doesn't exist in current scaffolds** — the unified command is now `yarn start` (runs frontend + backend together). Course material referencing `yarn dev`/`yarn start-backend` separately is from an older Backstage CLI generation.

### GitHub OAuth config (in progress at handoff — two bugs just fixed, untested)
- Location: `~/dev/backstage-app/backstage/app-config.local.yaml` (gitignored by the Backstage scaffold's own `.gitignore`, not part of this course repo — do not try to commit it here).
- **Bug 1 (fixed):** `auth.environment: devlepment` → typo, must exactly match the provider's environment key. Corrected to `development`.
- **Bug 2 (fixed):** the `- resolver: usernameMatchingUserEntityName` line was a bare list item directly under `development:`, not wired to anything. Backstage expects it nested under `signIn.resolvers`:
  ```yaml
  development:
    clientId: ${AUTH_GITHUB_CLIENT_ID}
    clientSecret: ${AUTH_GITHUB_CLIENT_SECRET}
    signIn:
      resolvers:
        - resolver: usernameMatchingUserEntityName
  ```
- **Not yet verified:** whether `AUTH_GITHUB_CLIENT_ID`/`AUTH_GITHUB_CLIENT_SECRET` env vars are actually set in the running container, whether the GitHub OAuth App's callback URL matches `http://localhost:3000` (or whatever `app.baseUrl` ends up being), and whether sign-in actually completes. **First thing to check/finish in a fresh session if OAuth is still the active task.**

### Backstage container / working setup (current, as of handoff)
- Running container: started via `docker run --rm -p 3000:3000 -ti -v ~/dev/backstage-app:/app -w /app node:22-bookworm-slim bash`, app scaffolded at `/app/backstage` (host path `~/dev/backstage-app/backstage/`). Container name at handoff: `happy_poitras` (Docker auto-name — will differ if recreated; find it with `docker ps --filter "ancestor=node:22-bookworm-slim"`).
- Both `NODE_EXTRA_CA_CERTS` and the CA cert copy are container-local — if the container is removed/recreated, redo the full CA-cert + toolchain + `app.listen.host` setup from scratch (nothing here persists outside the bind-mounted `~/dev/backstage-app` directory, which keeps the scaffolded app files but not container-level trust-store/env changes).
- Instructor's reference repo cloned locally this session for convenience: `~/dev/python-app` (from `github.com/ricardoandre97/python-app`). It has `catalog-info.yaml` and `mkdocs.yaml`/`docs/index.md` (placeholder TechDocs content, not lecture notes) — useful for Section 4/5 file-structure reference, not a source of lecture transcripts. No lecture/syllabus content exists anywhere in this session's local files or git history beyond the section-level checklist already in this repo's `README.md`.

---

## What is DONE / NOT done

### Done
- Everything from the lecture-60 kit (Section 2 complete, see that file).
- Backstage app scaffolded (`@backstage/create-app@latest`, Node 22) and **confirmed reachable in the Mac browser** at `http://localhost:3000` after the `app.listen.host: 0.0.0.0` fix.
- Two config bugs in the in-progress GitHub OAuth setup fixed (typo + missing `signIn.resolvers` nesting) — not yet tested end-to-end.
- README.md **not yet updated** for any Section 3 items — nothing has been committed to this repo for Section 3 yet. The Backstage work lives entirely outside this repo's tracked files.

### Not done
- GitHub OAuth sign-in — config fixed, client ID/secret + callback URL + actual sign-in flow untested.
- Plugins downloaded and added to frontend (Section 3 checklist item) — not started.
- Backstage authentication working (Section 3 checklist item) — blocked on OAuth above.
- Updating `README.md` checklist for Section 3 — hold until the user confirms specific items are actually done (per user instruction this session: don't update/commit until things are verified working).
- Investigating/deciding what to do about the always-on `rancher-desktop` k3s cluster eating ~5.4–6G of RAM — flagged, not acted on, needs the user's call.
- Rotating the exposed GitHub PAT (still outstanding from lecture-60 kit).
- Applying the corporate CA cert fix to `forge-idp`'s node (still outstanding, only needed if that cluster hits a registry TLS error).

---

## Preflight (run at start of fresh session)

```bash
# 0. VM egress check — same intermittent-drop issue as before
rdctl shell -- curl -sS -m 5 -o /dev/null -w "%{http_code}\n" https://github.com   # expect 200-ish
# If it times out:
osascript -e 'quit app "Rancher Desktop"'
open -a "Rancher Desktop"
# wait ~25-30s for docker info to respond, then re-run the curl check above

# 1. Cluster state (only devops-course should be running during coursework)
docker ps -a --filter "name=kind-control-plane" --filter "name=devops-course-control-plane" --format '{{.Names}}\t{{.Status}}'
docker stop kind-control-plane 2>/dev/null   # only if it came up — forge-idp stays stopped
kubectl config use-context kind-devops-course
kubectl get nodes
kubectl get pods -n argocd
kubectl get application python-app -n argocd
# If SYNC STATUS is Unknown/ComparisonError and egress is actually fine, force a refresh:
kubectl patch application python-app -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# 2. Memory sanity check (the undocumented rancher-desktop k3s cluster runs continuously)
rdctl shell -- free -h   # if "available" is under ~1G, expect flakiness in anything memory-hungry (npm/yarn native builds especially)

# 3. Backstage container — check if it's still running from last session
docker ps --filter "ancestor=node:22-bookworm-slim" --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
# If not running, recreate it:
#   docker run --rm -p 3000:3000 -ti -v ~/dev/backstage-app:/app -w /app node:22-bookworm-slim bash
# then inside: apt-get update && apt-get install -y python3 make g++ ca-certificates
# then redo the CA cert copy + NODE_EXTRA_CA_CERTS export (see "Corporate CA trust" above)
# then: cd backstage && yarn start   (app.listen.host: 0.0.0.0 is already saved in app-config.yaml on the host mount)

# 4. Git state
git -C /Users/diamondcj/dev/DevOps2PlatEng1 status
git -C /Users/diamondcj/dev/DevOps2PlatEng1 pull
git -C /Users/diamondcj/dev/DevOps2PlatEng1 log --oneline -5
```

---

## Kickoff instructions (fresh session executes these exactly)

1. Read this file fully.
2. Run the Preflight block above — fix Rancher Desktop networking FIRST if needed.
3. Read `README.md` to see current progress state (Section 3 checklist is still all unchecked — do not check anything off without the user confirming it's actually done).
4. Read `CLAUDE.md` for codebase context.
5. Check whether the Backstage container (`happy_poitras` or whatever it's since been renamed to) is still running and whether `yarn start` is still active inside it. If the user says they're still on OAuth, that's the live task — check `~/dev/backstage-app/backstage/app-config.local.yaml` matches the fixed version documented above before assuming it's still broken.
6. Tell the user: "Ready. I have context from the restart kit — Backstage is scaffolded and reachable at localhost:3000, OAuth config bugs are fixed but untested. What's the status on OAuth, or what lecture are you on now?"
7. Wait for the user — this is an interactive learning session. Help them as they work through each lecture. When they complete something and confirm it's working, update the README checklist and commit.

**Do not autonomously start implementing anything. The user drives the pace.**

---

## Sharp edges the executor will hit

- **`rm -rf` on host paths gets denied by a permission hook** for this session/user (observed twice — once on an empty scaffold dir, once as part of a chained command). Don't retry the same call; either ask the user to run it themselves, or find a non-destructive path forward. This is a standing environment constraint, not a one-off fluke.
- **Don't run competing `docker exec`/`yarn start` commands into a container the user already has an interactive shell open in** — collides on ports 3000/7007 (`EADDRINUSE`) and wastes a debug cycle. If the user has a live shell in the container, give them commands to run there instead of racing them with `docker exec`.
- **`WebSearch` 400s on this backend** — this session used `ask-codex "web search: ..."` successfully multiple times (Node version requirements, Backstage `app.listen.host` config) — keep using that pattern, it's reliable here.
- **The instructor's reference repo (`~/dev/python-app`) has no lecture content** — don't re-check it for syllabus text; it's code-structure reference only (Flask app, Helm chart, ArgoCD chart, CI/CD workflows, `catalog-info.yaml`, `mkdocs.yaml`). The only saved chapter/section checklist anywhere is this repo's own `README.md`.
- **Backstage's `app-config.local.yaml` is gitignored by design** (local secrets/overrides) — never try to commit it to this repo, and never paste its resolved secret values into this kit or any other committed file. Reference it by path and describe structure only, as done above.
