# RESTART KIT — DevOps2PlatEng1: Course Progress (mid-Section 6, lecture ~94, Software Templates)

**Created:** 2026-08-14, mid-Section 6 (Backstage Software Templates). About to start lecture #94 ("Hands On: Let's turn our DevOps project into a Backstage Software template").
**For:** a fresh session to continue hands-on course work alongside the user.
**Status:** Section 6 "Setup" and "Template Development" mechanics (parameters, steps, catalog wiring, render) all confirmed working end-to-end with a throwaway test file. The REAL conversion of python-app's actual source into the template has NOT started yet — that's lecture #94, about to begin.
**Approval gate:** user pasting the kickoff prompt below is the go — do not re-plan or re-ask.

---

## TL;DR

Udemy course repo (`cd1amond/DevOps2PlatEng1`, public). You help the user debug, write config/code, tick off README progress, commit, and push as they work through lectures. Co-pilot, not autonomous executor — wait for the user to say what lecture they're on.

Since the last kit (lecture 82), the user finished the plumbing for Section 6 Software Templates: created a dedicated templates repo, wired GitHub OAuth/integration, and proved the full scaffolder pipeline (parameters → fetch → publish → catalog register) works using a single throwaway test file as payload. **The actual DevOps project (python-app) has NOT yet been converted into the template** — a real skeleton was committed to the templates repo but that happened in a way the user flagged as premature; lecture #94 is exactly the "do this for real" step, not yet started.

**Do not re-plan, do not re-derive. Do not re-litigate decisions already made below.**

---

## Verified ground truth (trust these)

### GitHub OAuth — fixed this session, now genuinely correct
- Previous kit's "don't second-guess these values" note was **wrong** — the prior `clientId`/`clientSecret` in `app-config.local.yaml` had been swapped for garbage (`clientId: "cd1amond"` — a username, not an OAuth Client ID; `clientSecret` was a `ghp_...` Personal Access Token, not an OAuth App secret). This caused a hard 404 on `github.com/login/oauth/authorize`.
- **Fixed and confirmed working:** real values now in place — `clientId: "Ov23lipzreEPLN82HGtS"`, real `clientSecret` (in the gitignored `app-config.local.yaml`, not reproduced here). Sign-in tested end-to-end, including the expected "doesn't re-prompt after Backstage sign-out" behavior (that's normal OAuth — Backstage sign-out only clears the Backstage session, not the browser's live github.com session; not a bug).
- **Lesson for future debugging in this project:** don't trust a prior kit's "confirmed correct, don't second-guess" note over live evidence. If something 404s/errors, verify current file contents before assuming a past confirmation still holds.

### Backend startup bug — fixed
- `~/dev/backstage-app/backstage/packages/backend/src/index.ts` had `backend.add(import('@backstage/plugin-scaffolder-backend-module-github'))` registered **twice** — once correctly grouped under `// scaffolder plugin` near the top, once stray right before `backend.start()`. Caused `Module 'github' for plugin 'scaffolder' is already registered` and a full backend crash on startup.
- Fixed: removed the stray duplicate at the bottom. If this error resurfaces, check for a re-added duplicate before doing anything else.

### New GitHub repo for templates — created this session
- **`cd1amond/backstage-software-templates`** — created public (Backstage's scaffolder needs to fetch `template.yaml` via a `type: url` catalog location; public avoids needing a working GitHub integration token just to *read* templates — separate from the token needed to *publish* new repos, see below).
- Cloned locally to `~/dev/backstage-software-templates`.
- **Do not confuse with `~/dev/Class-backstage-software-templates`** — that's the instructor's reference clone (from `github.com/ricardoandre97/backstage-software-templates`). The previous kit's path note for the reference repo was slightly wrong (said `~/dev/backstage-software-templates`); corrected here. Two separate directories, two separate remotes.

### `app-config.local.yaml` catalog location — YAML bug fixed, template location added
- Added a `type: url` catalog location for the new Template entity, pointing at `https://github.com/cd1amond/backstage-software-templates/blob/main/python-app/template.yaml`, with `rules: [allow: [Template]]`.
- Hit a `YAMLParseError: A block sequence may not be used as an implicit map key` — root cause was a plain indentation typo (the new `- type: url` line had 3 spaces instead of the 4 spaces its sibling list items use). Fixed. **General lesson for this project:** these YAML errors' reported line/column point at where the parser gave up, not necessarily the actual bad line — check indentation on the lines just above.
- Backstage does **not** live-fetch `template.yaml` on every "Create" click — the catalog caches the Template entity from the last processed fetch of that `url` location. After pushing a `template.yaml` change to GitHub, either restart `yarn start` (reprocesses all locations on boot) or use the entity page's refresh icon in the Backstage UI (faster, no restart) before retrying.

### `publish:github` action — version-drift bug, fixed
- The scaffolder step for `publish:github` had an `allowedHosts` property in its `input:` block. **This exact block also exists in the instructor's reference `template.yaml`** — not a typo, a version mismatch. The currently-installed `@backstage/plugin-scaffolder-backend-module-github` rejects `allowedHosts` as an unrecognized property (strict schema validation); the course was recorded against an older version that accepted it.
- Fixed by removing `allowedHosts` from the step in `~/dev/backstage-software-templates/python-app/template.yaml` (commit `d158c24`, pushed). GitHub host resolution for `publish:github` comes from `integrations.github` config, not this input — the property wasn't needed at all.
- **If the reference repo's template.yaml is ever copied from again, strip `allowedHosts` from any `publish:github` step every time.**

### GitHub integration token — configured, deliberately reusing the flagged-for-rotation PAT
- `integrations.github[0].token` in `app-config.local.yaml` was `${GITHUB_TOKEN}` (unresolved — no such env var set anywhere in the container), causing `No token available for host: github.com` on the `publish:github` step.
- **Fixed by explicit user decision:** replaced with the literal value of the same `<the-ghp_-PAT-flagged-for-rotation-since-lecture-60>` PAT that was previously (incorrectly) sitting in the OAuth `clientSecret` field. User's stated reasoning: "they will all get rotated together and this class is short lived" — a deliberate, informed tradeoff. **Do not re-raise this as a new concern** — it's the same standing rotation item from earlier kits, not a new leak. This token now lives in exactly one place in the gitignored `app-config.local.yaml` (the `integrations.github` block); the bogus copy in `clientSecret` was already overwritten with the real OAuth secret.
- This PAT needs `repo` scope to create GitHub repos via the scaffolder. Confirmed working — the test template run successfully created `cd1amond/python-app-1` on GitHub.

### Backend session/signals WebSocket errors after restart — not a bug, expected
- Saw `AuthenticationError: Failed user token verification` on the `signals` plugin's WebSocket right after a `yarn start` restart.
- Root cause: no `backend.database` config anywhere (checked `app-config.yaml` and `app-config.local.yaml` — neither sets one), so Backstage defaults to an **in-memory SQLite DB** in dev mode. Every backend restart regenerates the auth signing keys from scratch; any session token issued before the restart becomes invalid.
- **Fix is always the same: sign out and back in (or hard-refresh the browser tab) after any backend restart.** Not worth persisting the dev DB for a short-lived course setup — don't suggest it unless asked.

### First template deployment — succeeded, but was only a proof-of-path test
- The end-to-end scaffolder pipeline (parameters → `fetch:template` → `publish:github` → `catalog:register`) was proven working using a **single throwaway test file**, not the real python-app source. This created a real repo on GitHub: `cd1amond/python-app-1`. It's a genuine leftover test artifact — harmless, but if a fresh session sees an unexplained extra repo, that's why.
- Immediately after, a commit (`cfc106a`, "Adds python app software template") to `backstage-software-templates` copied the **real** DevOps2PlatEng1 files into `python-app/template/` — `src/app.py`, `Dockerfile`, `charts/`, `k8s/`, `requirements.txt`, `.github/workflows/cicd.yaml`, `docs/index.md`, `docs/mkdocs.yaml`, with Helm chart paths parameterized via `${{values.app_name}}`. **This commit also accidentally swept in all 4 of DevOps2PlatEng1's own `docs/plans/*-RESUME.md` restart-kit files** into `python-app/template/docs/plans/` — caught and removed this session (commit `7703613`, pushed). `fetch:template` copies the whole source directory verbatim, so those stale kits would otherwise have been baked into every future generated repo.
- **The user explicitly clarified this `cfc106a` copy was premature/exploratory** — "we only did a single mini test text file to show the whole path, now we're converting the python src from before." Lecture #94 is the actual from-scratch walkthrough of that conversion. Treat the current `python-app/template/` contents as a rough draft to be redone properly during #94, not as finished work — README reflects this (see below).

---

## What is DONE / NOT done

### Done
- Everything from the lecture-82 kit (Sections 1-5 complete, catalog/TechDocs working), plus:
- GitHub OAuth actually fixed and verified working (not just "assumed still working" — this kit's predecessor was wrong about that).
- Backend duplicate-module crash fixed.
- `backstage-software-templates` repo created, cloned, and is the active home for Section 6 template work.
- Section 6 **Setup** (all 3 items) and **Input parameters defined** — committed (`db6ea64`).
- Section 6 **Template steps defined**, **`catalog-info.yaml` wired into template**, **Template renders successfully in Backstage** — committed (`a73aef2`). These are legitimately done: proven via the test-file run.
- Stray restart-kit files removed from the template's `docs/plans/` — committed and pushed (`7703613` in `backstage-software-templates`).
- README correction: **"DevOps project rewritten as a Backstage Software Template" un-checked** — committed and pushed (`0a237b2`) after the user clarified only a test file had gone through the pipeline, not the real source.

### Not done
- **The real python-app → template conversion (lecture #94) — not started.** The `cfc106a` commit's copy of real files into `python-app/template/` should be treated as a rough draft; expect the user to redo/rework this properly during #94.
- **GitHub Orgs & Distributed Builds** section — nothing started (GitHub org config, distributed builds, repo properties from Backstage Actions).
- **ArgoCD Integration** section — nothing started (template-driven ArgoCD automation, full end-to-end repo-create → ArgoCD-sync test).
- **CI/CD view customization** — not started.
- Leftover test repo `cd1amond/python-app-1` on GitHub — harmless, not cleaned up, no action needed unless the user asks.
- Rotating the `<the-ghp_-PAT-flagged-for-rotation-since-lecture-60>` PAT — still outstanding, now used in one place (`integrations.github.token`) instead of two. User has explicitly deferred this to a broader planned rotation — don't nag.
- Everything else already flagged as outstanding in the lecture-82 kit (forge-idp CA cert fix, etc.) — unchanged, not touched this session.

---

## Preflight (run at start of fresh session)

```bash
# 1. kubectl context sanity
kubectl config get-contexts
# Expect current context: kind-devops-course. If it's ever an arn:aws:eks:* context,
# STOP — do not run further kubectl commands until confirmed safe (see lecture-82 kit
# for the full incident writeup; this is a standing concern, not hypothetical).

# 2. Cluster state (only devops-course should be running during coursework)
docker ps -a --filter "name=kind-control-plane" --filter "name=devops-course-control-plane" --format '{{.Names}}\t{{.Status}}'
kubectl get nodes
kubectl get application python-app -n argocd   # expect Synced/Healthy

# 3. Confirm rancher-desktop k3s stack is still off (user's deliberate choice)
rdctl list-settings 2>/dev/null | python3 -c "import json,sys; print('kubernetes.enabled =', json.load(sys.stdin)['kubernetes']['enabled'])"
# Expect: False.

# 4. Backstage container
docker ps --filter "name=backstage-course" --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
# If not running: docker start backstage-course. The user runs `yarn start` themselves
# inside it interactively — don't race it with a competing docker exec/yarn start
# (EADDRINUSE). If OAuth/websocket errors show up right after a restart, that's the
# in-memory-DB session-invalidation behavior described above — sign out/in, don't chase it as a bug.

# 5. Git state — TWO repos matter now
git -C /Users/diamondcj/dev/DevOps2PlatEng1 status
git -C /Users/diamondcj/dev/DevOps2PlatEng1 pull
git -C /Users/diamondcj/dev/DevOps2PlatEng1 log --oneline -5

git -C ~/dev/backstage-software-templates status
git -C ~/dev/backstage-software-templates pull
git -C ~/dev/backstage-software-templates log --oneline -5
```

---

## Kickoff instructions (fresh session executes these exactly)

1. Read this file fully.
2. Run the Preflight block above.
3. Read `README.md` in `DevOps2PlatEng1` — Sections 1-5 fully checked; Section 6 "Setup" + "Input parameters defined" + "Template steps defined" + "`catalog-info.yaml` wired into template" + "Template renders successfully in Backstage" checked; "DevOps project rewritten as a Backstage Software Template" **unchecked** (that's the next real work).
4. Read `CLAUDE.md` for codebase context.
5. If the user says they're starting or continuing lecture #94, check `~/dev/backstage-software-templates/python-app/template/` current contents first — it already has a rough-draft copy of the real python-app files (from commit `cfc106a`) that the user said was premature/exploratory. Don't assume it's final; ask what state they want it treated as before editing.
6. Tell the user: "Ready. Section 6 templating mechanics are proven working end-to-end with a test file; the real python-app → template conversion (lecture #94) hasn't been done for real yet. There's a rough draft already in `backstage-software-templates/python-app/template/` from an earlier exploratory commit. What's the status, or what lecture are you on?"
7. Wait for the user — interactive learning session, they drive the pace. Update the README checklist and commit only when they confirm something's actually done. **This user has explicitly said commits/pushes of `docs/plans/` restart kits ARE wanted in this specific repo** (it's personal class work, not a work-controlled repo) — the global CLAUDE.md guardrail requiring confirmation before committing `docs/plans/` does not need to be re-asked here; that confirmation was already given for this repo.

**Do not autonomously start implementing anything. The user drives the pace.**

---

## Sharp edges the executor will hit

- **Don't trust a prior kit's "don't second-guess this config" note over live evidence.** This session's OAuth bug proved that wrong once already.
- **Two git repos are now in play**, not one — `DevOps2PlatEng1` (course progress, README, this kit) and `~/dev/backstage-software-templates` (the actual template content). Check status/log on both.
- **`fetch:template` copies its source directory verbatim** — any stray file dropped into `python-app/template/` (like the restart-kit incident) gets baked into every future generated repo. Sanity-check directory contents before committing template changes.
- **`app-config.local.yaml` catalog `url` locations are cached, not live-fetched per Create click.** After pushing a template.yaml change, restart the backend or use the entity page's refresh icon before retrying — a stale cached version will silently reproduce an already-fixed error.
- **YAML errors' reported line/column point at parser confusion, not necessarily the actual bad line** — check indentation just above the reported location first.
- **`allowedHosts` on a `publish:github` step is version-drift cruft from the course material** — strip it every time it's copied from the reference repo.
- **The `<the-ghp_-PAT-flagged-for-rotation-since-lecture-60>` PAT is deliberately live in `integrations.github.token`** (gitignored file, not committed) — don't flag as a new leak, don't suggest rotating it ahead of the user's own planned sweep.
- **`rm -rf` on host paths gets denied by a permission hook** for this session/user — use `git rm` for tracked files instead, or ask the user.
- **Don't race the user's interactive `yarn start` shell** with a competing `docker exec`/`yarn start` — EADDRINUSE.
- **`WebSearch` 400s on this backend** — use `ask-codex "web search: ..."` if research is needed.
- **This repo's convention: commit and push `docs/plans/*-RESUME.md` kits directly** — the user has explicitly authorized this for `DevOps2PlatEng1` specifically, overriding the general CLAUDE.md guardrail that normally requires asking first.
