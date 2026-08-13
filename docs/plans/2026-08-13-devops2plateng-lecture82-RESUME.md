# RESTART KIT — DevOps2PlatEng1: Course Progress (Section 6 starting, lecture ~82, Software Templates)

**Created:** 2026-08-13, start of Section 6 (Backstage Software Templates). Sections 1-5 fully complete and committed.
**For:** a fresh session to continue hands-on course work alongside the user.
**Status:** Section 5 (TechDocs) confirmed working. Section 6 just starting — no template work done yet in this repo. This is a learning/lab session, not a code-generation task.
**Approval gate:** user pasting the kickoff prompt below is the go — do not re-plan or re-ask.

---

## TL;DR

Udemy course repo (`cd1amond/DevOps2PlatEng1`, **now public** — see below). You help the user debug, write config/code, tick off README progress, commit, and push as they work through lectures. You are a co-pilot, not an autonomous executor. Wait for the user to tell you what lecture they're on.

Since the last kit (lecture 68), the user finished Sections 3-5 (Backstage OAuth, Catalog, TechDocs) and is starting Section 6 (Software Templates, lecture ~82). This session had a **major incident** (disabling Rancher Desktop's built-in Kubernetes crashed the VM and destroyed the Backstage dev container) that is now fully recovered — details below. Backstage itself is **not** part of this git repo — it's a separate scaffold at `~/dev/backstage-app/backstage/`.

**Do not re-plan, do not re-derive. Do not re-litigate decisions already made below.**

---

## Verified ground truth (trust these)

### Repo visibility — DevOps2PlatEng1 is now PUBLIC (deliberate, user decision)
- Changed from private to public this session (`gh repo edit ... --visibility public --accept-visibility-change-consequences`) specifically so Backstage's catalog-import could read `catalog-info.yaml` via `raw.githubusercontent.com` without needing a `GITHUB_TOKEN` configured in the backend.
- **Before this**, a full git history scan (`git log --all -p | grep -oE 'ghp_...|github_pat_...'`) found no PAT-shaped strings in `DevOps2PlatEng1`'s history — the repo itself is clean.
- User stated they have **a sweeping key rotation planned soon** — this was a known, accepted tradeoff, not an oversight. Don't re-raise it as a new concern.
- The **exposed-PAT-needs-rotating item from the lecture-60 kit is still outstanding** and now more relevant given the public repo — don't nag about it further, just don't assume it's been rotated.

### GitHub OAuth config — now uses literal values in app-config.local.yaml (deliberate)
- `~/dev/backstage-app/backstage/app-config.local.yaml` — the `auth.providers.github.development.clientId`/`clientSecret` fields hold **literal string values**, not `${AUTH_GITHUB_CLIENT_ID}`/`${AUTH_GITHUB_CLIENT_SECRET}` env-var references anymore. This was a deliberate simplification (this file is gitignored by the Backstage scaffold, not tracked anywhere) so OAuth survives container recreation without re-exporting env vars each time.
- **User confirmed the current values are correct and deliberate** — do not second-guess the `clientId`/`clientSecret` format or suggest they look wrong. This was already raised and settled this session.
- **Never read this file's contents back into a message that gets committed anywhere.** It is outside this git repo (not part of `DevOps2PlatEng1`), but treat its contents as sensitive on principle — never echo the actual values into any file that could be committed/pushed, especially now that the repo is public.
- Sign-in via GitHub OAuth was confirmed working end-to-end earlier this session (before the incident below), using a callback URL of exactly `http://localhost:7007/api/auth/github/handler/frame` on the GitHub OAuth App side. That callback URL config lives on GitHub's side (OAuth App settings), not in any local file — it survived the container recreation.

### MAJOR INCIDENT this session — Rancher Desktop VM crash, now recovered
- User asked to disable the always-on `rancher-desktop` k3s cluster (hosting ArgoCD/Keycloak/Postgres/Flux/external-secrets — flagged as a RAM hog in the lecture-68 kit) via `rdctl set --kubernetes.enabled=false`.
- This **crashed the Rancher Desktop VM** (went to "Broken" state). Recovery required: `osascript -e 'quit app "Rancher Desktop"'` then `open -a "Rancher Desktop"`, then the user manually confirmed the app relaunched (a GUI admin-password prompt may be needed on relaunch — headless `rdctl start`/`open -a` alone did not bring it back; needed the user's own GUI interaction).
- **During the crash, `kubectl`'s current-context briefly switched to a real AWS EKS production cluster** (`arn:aws:eks:us-east-1:963013857520:cluster/forge-prod-cluster` — NOT anything local, a genuine cross-account prod cluster). No command was ever run against it. Context was immediately unset (`kubectl config unset current-context`) and later re-set safely to `kind-devops-course`. **If you ever see `kubectl config get-contexts` list `forge-test-cluster`/`forge-dev-cluster`/`forge-prod-cluster` AWS EKS entries, do not run any kubectl command until you've explicitly confirmed the current context — these are real cross-account infrastructure, not course-related.**
- **Outcome after recovery:**
  - `rancher-desktop` k3s stack: confirmed **still disabled** (`rdctl list-settings` → `kubernetes.enabled: false`), no pods running there. This is what the user wanted — don't re-enable without being asked.
  - Both `devops-course-control-plane` and `kind-control-plane` (forge-idp) came back up on VM restart — `kind-control-plane` was stopped again afterward (only one kind cluster should run at a time; `forge-idp` never delete).
  - The old Backstage container (`happy_poitras`, later `loving_nightingale`) was started with `--rm` and did **not survive** the crash — fully deleted, not just stopped.
  - `~/dev/backstage-app` is a bind mount, so all app files (auth fixes, catalog entities, app-config) survived on disk untouched. Only container-level state (CA cert trust, `NODE_EXTRA_CA_CERTS`, installed toolchain/pip packages) was lost and had to be redone.

### Current Backstage dev container setup (rebuilt this session, current as of handoff)
- **New container name: `backstage-course`** (NOT `happy_poitras`/`loving_nightingale` from earlier kits — those are gone). Created **without** `--rm` this time, deliberately, so a future VM hiccup leaves it stoppable/restartable via `docker start backstage-course` instead of destroying it.
- Created from bare `node:22-bookworm-slim` (not yet from the new `backstage-dev` image below — that image was built *after* this container already existed). Currently running with: CA cert trusted, `NODE_EXTRA_CA_CERTS` set, `python3 make g++` toolchain, `python3-pip` + `mkdocs-techdocs-core` (pulls in `mkdocs`) all installed manually via `docker exec`. Verified: `mkdocs --version` → 1.6.1, TLS trust to `nodejs.org` verified via Node's `https` module.
- Ports: `0.0.0.0:3000->3000`, `0.0.0.0:7007->7007`. Bind mount: `~/dev/backstage-app:/app`.
- **A new `Dockerfile.dev` now exists at `~/dev/backstage-app/Dockerfile.dev`** (not part of any git repo — `~/dev/backstage-app` has no `.git`), building an image tagged `backstage-dev` that bakes in the toolchain + `mkdocs-techdocs-core` + `NODE_EXTRA_CA_CERTS` env var. **This is for the NEXT time the container needs recreating** — it has not been applied to the currently-running `backstage-course` container. To recreate using it:
  ```bash
  docker run -d --name backstage-course -p 3000:3000 -p 7007:7007 \
    -v ~/dev/backstage-app:/app backstage-dev
  ```
  Then still manually: copy the Leidos CA cert in and run `update-ca-certificates --fresh` (deliberately left out of the Dockerfile — corp-network-specific, documented as a comment at the top of `Dockerfile.dev`).
- `packages/backend/Dockerfile` (the scaffold's **production** build Dockerfile, different artifact entirely) was separately edited to add `python3-pip` + `pip3 install mkdocs-techdocs-core` for the local TechDocs generator — this only matters if/when that image is actually built (`yarn build-image`), which hasn't happened this session.

### TechDocs — confirmed working
- `techdocs.generator.runIn: 'local'` is set in `app-config.local.yaml` (not the default `'docker'` mode) — this means the **backend process itself** shells out to `mkdocs` directly, so `mkdocs` must be installed wherever the backend runs (the dev container), not just in a production image. This is now true for `backstage-course` (see above).
- Section 5 checklist items (docs/ folder, mkdocs.yml, TechDocs plugin, docs rendering) all confirmed done by the user, committed in `0b542aa`.

### Catalog — confirmed working after a reimport
- `catalog-info.yaml` (repo root) defines `Component`/`python-app`, no explicit namespace (defaults to `default`) — matches `localhost:3000/catalog/default/component/python-app`.
- First registration attempt via `/catalog-import` failed with `Entity not found` even though the file was correctly pushed — root cause was the repo being private at the time (raw.githubusercontent.com 404s on private repos without a token). **Making the repo public did not automatically fix an already-attempted import** — the instructor's own material flagged a "may need to reimport" step. User confirmed: **reimporting through the `/catalog-import` wizard (Analyze → Import, not just typing the URL) fixed it.**
- Lesson banked from this: typing a raw entity URL directly into the browser address bar (e.g. `/catalog/default/component/python-app`) only works if the entity was actually registered via the import wizard's full flow — Analyze alone doesn't register anything.

### VS Code YAML schema false positives — fixed globally
- `redhat.vscode-yaml`'s built-in Schema Store auto-detection matches files literally named `template.yaml`/`template.yml` to the **AWS SAM CloudFormation schema** regardless of content, and `mkdocs.yaml`/`mkdocs.yml` to a generic MkDocs schema that doesn't know Backstage's `techdocs-core` plugin. Both caused false "Property X is not allowed" errors on legitimate Backstage files.
- Fixed in **global VS Code user settings** (`~/Library/Application Support/Code/User/settings.json`, not workspace-local — this problem follows the filename pattern across any repo):
  ```json
  "yaml.disableSchemaDetection": ["**/template.yaml", "**/template.yml", "**/mkdocs.yaml", "**/mkdocs.yml"]
  ```
- User also installed `ms-kubernetes-tools.vscode-kubernetes-tools` this session (their own initiative, looking for "yaml support" for Kubernetes/Helm). No confirmed conflicts from it — a suspected conflict with `catalog-info.yaml` (whose `apiVersion`/`kind`/`metadata`/`spec` shape mimics Kubernetes resources) turned out to be a non-issue after a reload; the user reported it "looks clean." Don't re-investigate unless something actually flags again.

### Reference repos (instructor material, cloned this session)
- `~/dev/backstage-software-templates` (from `github.com/ricardoandre97/backstage-software-templates`) — contains `python-app/template.yaml` + `python-app/template/` (full skeleton: `catalog-info.yaml`, `Dockerfile`, `charts/`, `docs/`, `k8s/`, `mkdocs.yaml`, `requirements.txt`, `src/`). This is the reference for exactly what Section 6 has the user build — expect them to copy/adapt from here rather than typing from scratch.
- `~/dev/python-app` (from lecture-68 kit, still present) — Flask app + Helm chart reference, structure-only, no lecture content.
- Neither reference repo has actual lecture/syllabus text anywhere — this repo's own `README.md` remains the only saved progress checklist.

---

## What is DONE / NOT done

### Done
- Everything from the lecture-68 kit, plus:
- Sections 3, 4, 5 all confirmed complete and committed (`28c5ca2`, `0b542aa`).
- GitHub OAuth sign-in confirmed working end-to-end (pre-incident; config should still be valid post-recovery, but **not re-verified after the container rebuild** — see Preflight).
- Catalog registration confirmed working after reimport.
- TechDocs confirmed working (local mkdocs generator).
- `rancher-desktop` k3s stack disabled per user request, confirmed still off after VM recovery.
- New `backstage-dev` Dockerfile image built and verified (toolchain + mkdocs present) — ready for next container recreation, not yet applied to the running container.
- VS Code YAML schema false positives fixed globally.
- Reference repo for Section 6 cloned.

### Not done
- **Section 6 (Software Templates) — nothing started yet.** Checklist: Backstage Actions plugin, GitHub integration, dedicated templates repo, input parameters, template steps, `catalog-info.yaml` wired into template, DevOps project rewritten as a template, template renders successfully, GitHub org config, distributed builds, repo properties from Backstage Actions, ArgoCD automation via templates, full end-to-end test, CI/CD view customization.
- **OAuth sign-in has not been re-tested since the container rebuild** (mid-incident-recovery) — the config in `app-config.local.yaml` should still be correct (survived on the bind mount), but confirm sign-in actually still works before assuming it does.
- Rotating the exposed GitHub PAT — still outstanding, now more pressing given the repo is public. User has stated a broader key rotation is planned; don't nag, but don't assume done either.
- Applying the corporate CA cert fix to `forge-idp`'s node — still outstanding (only matters if that cluster hits a registry TLS error; it's currently stopped).
- Recreating `backstage-course` from the new `backstage-dev` image — not needed unless the container is lost again; current one is working fine as-is.

---

## Preflight (run at start of fresh session)

```bash
# 0. VM egress check
rdctl shell -- curl -sS -m 5 -o /dev/null -w "%{http_code}\n" https://github.com   # expect 200
# If it times out, restart Rancher Desktop (quit + relaunch via GUI — headless open -a alone
# was NOT sufficient to recover from the crash this session; watch for a GUI password prompt).

# 1. kubectl context sanity — CRITICAL after last session's incident
kubectl config get-contexts
# If current context is anything under arn:aws:eks:* (forge-test/dev/prod-cluster), DO NOT
# run any further kubectl command — unset it first: kubectl config unset current-context
# Then explicitly: kubectl config use-context kind-devops-course

# 2. Cluster state (only devops-course should be running during coursework)
docker ps -a --filter "name=kind-control-plane" --filter "name=devops-course-control-plane" --format '{{.Names}}\t{{.Status}}'
docker stop kind-control-plane 2>/dev/null   # forge-idp — only if it came up, stays stopped
kubectl get nodes
kubectl get application python-app -n argocd   # expect Synced/Healthy

# 3. Confirm rancher-desktop k3s stack is still off (user's deliberate choice)
rdctl list-settings 2>/dev/null | python3 -c "import json,sys; print('kubernetes.enabled =', json.load(sys.stdin)['kubernetes']['enabled'])"
# Expect: False. Do not re-enable without being asked.

# 4. Backstage container
docker ps --filter "name=backstage-course" --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
# If not running: docker start backstage-course (do NOT recreate unless it's gone —
# check `docker ps -a` first; only rebuild from backstage-dev image if truly destroyed)

# 5. Git state
git -C /Users/diamondcj/dev/DevOps2PlatEng1 status
git -C /Users/diamondcj/dev/DevOps2PlatEng1 pull
git -C /Users/diamondcj/dev/DevOps2PlatEng1 log --oneline -5
```

---

## Kickoff instructions (fresh session executes these exactly)

1. Read this file fully.
2. Run the Preflight block above — the kubectl-context check (step 1) is not optional given last session's incident.
3. Read `README.md` — Sections 1-5 should show fully checked; Section 6 unchecked (start of this kit's work).
4. Read `CLAUDE.md` for codebase context. Note: the *lecture-68* restart kit (`docs/plans/2026-08-12-devops2plateng-lecture68-RESUME.md`) describes the `rancher-desktop` k3s stack as something to ask before touching — that guidance is superseded by this kit (it's already been disabled, deliberately, per direct user instruction this session). Don't re-ask about it.
5. If the user says they're working on Section 6, check whether `backstage-course` is running and whether `yarn start` is active inside it. If OAuth sign-in comes up, don't assume it's broken — test it first; it likely still works since `app-config.local.yaml` survived on the bind mount.
6. Tell the user: "Ready. I have context from the restart kit — Backstage is running in `backstage-course`, Sections 1-5 are done, and we're starting Section 6 (Software Templates). There's a reference repo cloned at `~/dev/backstage-software-templates` if useful. What's the status, or what lecture are you on?"
7. Wait for the user — interactive learning session, they drive the pace. Update the README checklist and commit only when they confirm something's actually done.

**Do not autonomously start implementing anything. The user drives the pace.**

---

## Sharp edges the executor will hit

- **Never write secret values from `app-config.local.yaml` into any file that gets committed** — the repo is now public. This applies even more strictly than before.
- **`kubectl` context safety is now a standing concern, not hypothetical** — this session proved a VM crash can silently rewrite the current context to a real AWS EKS production cluster. Always sanity-check `kubectl config get-contexts` after any Rancher Desktop restart/crash before running further kubectl commands.
- **Don't touch `rancher-desktop`'s Kubernetes setting** (`rdctl set --kubernetes.enabled=...`) without direct instruction — the last time this was done, it crashed the VM. If asked again, warn about this history first.
- **`rm -rf` on host paths gets denied by a permission hook** for this session/user — don't retry the same call; ask the user or find a non-destructive path.
- **Don't race the user's interactive container shell** with competing `docker exec`/`yarn start` — same EADDRINUSE issue as before.
- **`WebSearch` 400s on this backend** — use `ask-codex "web search: ..."` (used successfully this session for the VS Code YAML schema issue).
- **`backstage-dev` image exists but is not yet applied** to the running container — don't assume the current `backstage-course` container was built from it; it wasn't (manual setup, done before the image existed).
