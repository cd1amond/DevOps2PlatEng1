# RESTART KIT — DevOps2PlatEng1: Course Progress (Section 6, lecture #99, GitHub Orgs)

**Created:** 2026-08-20, paused at the start of lecture #99 ("Configure GitHub organizations - I"). User is catching up on course prerequisites the class skipped over before continuing.
**For:** a fresh session to continue hands-on course work alongside the user.
**Status:** Section 6 "Setup" and "Template Development" fully done — including the *real* python-app conversion (lecture #94-98), confirmed and cleaned up this session. Lecture #99 (GitHub Orgs & Distributed Builds) has NOT started — no lab work done yet, paused for prerequisite catch-up.
**Approval gate:** user pasting the kickoff prompt below is the go — do not re-plan or re-ask.

---

## TL;DR

Udemy course repo (`cd1amond/DevOps2PlatEng1`, public). You help the user debug, write config/code, tick off README progress, commit, and push as they work through lectures. Co-pilot, not autonomous executor — wait for the user to say what lecture they're on.

Since the last kit (lecture 94, mid-session), two things happened:
1. **Confirmed the real python-app→template conversion (lecture #94-98) was actually done** — the previous kit had this flagged as "not started," but that was wrong. `git show --stat cfc106a` in `backstage-software-templates` proved the real files (`src/app.py`, `Dockerfile`, parameterized Helm chart, `k8s/`, `requirements.txt`, `docs/`, CI workflow) were genuinely copied and templated, not just a test file. Screenshot evidence also showed the "Python flask template" live and browsable in Backstage's Create UI.
2. **Cleaned up two file issues found during that verification** (commit `c59ac97` in `backstage-software-templates`): removed a stray `test.txt` left over from the earlier pipeline-proof run (would've been baked into every scaffolded repo via `fetch:template`'s verbatim copy), and fixed `runnerdeployment.yaml`'s hardcoded `repository: cd1amond/DevOps2PlatEng1` to use `${{values.app_name}}` instead (every generated repo's self-hosted-runner snippet was pointing at the *original* repo, not itself).
3. README updated to check off "DevOps project rewritten as a Backstage Software Template" (commit `0625d47` in `DevOps2PlatEng1`).

**User then paused before starting #99's actual lab work** — reason given: "the class is moving past pieces which have been left out so the gaps leave me flailing to catch up with prereqs mostly, just need to come back to it later." This is NOT a technical bug to fix — it's the user needing to backfill course prerequisites (GitHub Orgs concepts/setup) before attempting the lecture's steps. No specific missing prereq was identified this session; ask the user what they've caught up on when they resume.

**Do not re-plan, do not re-derive. Do not re-litigate decisions already made below.**

---

## Verified ground truth (trust these)

### The real python-app→template conversion IS done — verified via git history, not assumed
- The lecture-94 kit's own text called `cfc106a` (in `backstage-software-templates`) "premature/exploratory," based on the user's framing at the time. This session re-verified that framing was too pessimistic: `git show --stat cfc106a` shows a genuine 27-file diff copying real project source into `python-app/template/`, with Helm chart paths parameterized via `${{values.app_name}}`. Cross-checked file-by-file against the actual `DevOps2PlatEng1` repo — every file present (including `charts/argocd/values-argo.yaml` and `runnerdeployment.yaml`) has a real counterpart in the source repo. Only `test.txt` was genuinely stray (didn't exist in the source repo at all — leftover from the earlier single-file pipeline-proof run).
- **Lesson for future sessions in this project:** a prior kit's "user said X was premature" note can be an accurate snapshot of what the user believed *at that moment*, not a durable fact — re-verify against git evidence before propagating the same "not done" claim forward. This is the second time in this project's kit chain that a "don't second-guess this" note turned out stale (see lecture-94 kit's OAuth lesson for the first).

### `runnerdeployment.yaml` parameterization — fixed this session
- Was: `repository:  cd1amond/DevOps2PlatEng1` (hardcoded, wrong for any generated repo other than the original).
- Now: `repository:  cd1amond/${{values.app_name}}` — matches the templating convention already used elsewhere in `python-app/template/` (e.g. the old `test.txt`, the parameterized chart directory name).
- This file is a `kubectl apply` heredoc snippet, not something any scaffolder step actually executes automatically — it's reference material that rides along in the generated repo. No template.yaml step change was needed, just the file content.

### Environment state at pause — both kind clusters are stopped, Backstage is up
- `backstage-course` container: **running**, up ~23h (was stopped last kit; now confirmed up — no action needed, don't restart it).
- Both `kind-control-plane` and `devops-course-control-plane`: **both exited** (137, ~23h ago) — this is different from every prior kit, where one of the two was always running. Neither cluster is currently up. `kubectl config current-context` still shows `kind-devops-course` but that container is stopped — commands will fail until the node container is started.
- This is expected idle-machine state, not a bug — just don't assume either cluster is live without checking first.

---

## What is DONE / NOT done

### Done
- Everything from the lecture-94 kit, plus:
- Real python-app→template conversion (lecture #94-98) — confirmed genuinely complete via git history, not just UI screenshot. Committed: `cfc106a`, `7703613` (stray restart-kit cleanup, done last kit), `c59ac97` (this session's test.txt removal + runnerdeployment.yaml fix), all in `backstage-software-templates`.
- README: "DevOps project rewritten as a Backstage Software Template" checked off — `DevOps2PlatEng1` commit `0625d47`.

### Not done
- **Lecture #99 ("Configure GitHub organizations - I") — not started at all.** No lab steps attempted. Paused for prerequisite catch-up.
- **GitHub Orgs & Distributed Builds** section — nothing started (GitHub org config, distributed builds, repo properties from Backstage Actions). Same three unchecked README items as every prior kit: GitHub organization configured; Distributed builds on GitHub Orgs; Repository properties configurable from Backstage Actions.
- **ArgoCD Integration** section — nothing started.
- **CI/CD view customization** — not started.
- Leftover test repo `cd1amond/python-app-1` on GitHub — still harmless, still not cleaned up, no action needed unless the user asks.
- Rotating the flagged PAT (still live in `integrations.github.token` in the gitignored `app-config.local.yaml`) — still outstanding, still explicitly deferred by the user. Don't nag.
- Everything else already flagged as outstanding in prior kits (forge-idp CA cert fix, etc.) — unchanged, not touched this session.

---

## Preflight (run at start of fresh session)

```bash
# 1. kubectl context sanity
kubectl config get-contexts
# Expect current context: kind-devops-course. If it's ever an arn:aws:eks:* context,
# STOP — do not run further kubectl commands until confirmed safe (standing concern
# from the lecture-82 kit's incident writeup).

# 2. Cluster state — AT PAUSE, BOTH kind containers were stopped (unusual, see above)
docker ps -a --filter "name=kind-control-plane" --filter "name=devops-course-control-plane" --format '{{.Names}}\t{{.Status}}'
# If devops-course-control-plane is exited, the user needs to start it before any
# kubectl/ArgoCD work: docker start devops-course-control-plane
kubectl get nodes
kubectl get application python-app -n argocd   # expect Synced/Healthy once cluster is up

# 3. Confirm rancher-desktop k3s stack is still off (user's deliberate choice)
rdctl list-settings 2>/dev/null | python3 -c "import json,sys; print('kubernetes.enabled =', json.load(sys.stdin)['kubernetes']['enabled'])"
# Expect: False.

# 4. Backstage container — was UP at pause, confirm still true
docker ps --filter "name=backstage-course" --format '{{.Names}}\t{{.Status}}\t{{.Ports}}'
# If not running: docker start backstage-course. User runs `yarn start` themselves
# inside it interactively — don't race it with a competing docker exec/yarn start
# (EADDRINUSE).

# 5. Git state — TWO repos matter
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
3. Read `README.md` in `DevOps2PlatEng1` — Sections 1-5 fully checked; Section 6 "Setup" + all "Template Development" items (including the real conversion) checked; "GitHub Orgs & Distributed Builds" through "CI/CD View" all unchecked — that's the next real work.
4. Read `CLAUDE.md` for codebase context.
5. Ask the user: "Ready. Lecture #99 (Configure GitHub organizations - I) hasn't been started — you paused last time to catch up on prerequisites the course skipped over. Are you caught up and ready to start #99, or is there something specific you want to work through first?"
6. Wait for the user — interactive learning session, they drive the pace. Update the README checklist and commit only when they confirm something's actually done. **This user has explicitly authorized committing/pushing `docs/plans/` restart kits in this specific repo** (personal class work, not work-controlled) — the global CLAUDE.md guardrail requiring confirmation before committing `docs/plans/` does not need to be re-asked here.

**Do not autonomously start implementing anything. The user drives the pace.**

---

## Sharp edges the executor will hit

- **Don't trust a prior kit's "user said this was premature/not done" note without re-checking git evidence.** This session already found one case of this in this same project — a status claim from the previous kit ("real conversion NOT started") turned out to be stale; the user had actually done it, just hadn't recorded it. Verify against `git log`/`git show` before propagating a "not done" claim forward again.
- **Two git repos are in play** — `DevOps2PlatEng1` (course progress, README, this kit) and `~/dev/backstage-software-templates` (the actual template content). Check status/log on both.
- **`fetch:template` copies its source directory verbatim** — any stray file dropped into `python-app/template/` gets baked into every future generated repo. This has now happened twice (restart-kit files, then `test.txt`) — sanity-check directory contents against the real source repo before committing template changes.
- **Both kind cluster containers were stopped at pause time** — unusual vs. every prior kit (one was always running). Don't assume the cluster is up; check first.
- **`allowedHosts` on a `publish:github` step is version-drift cruft from the course material** — strip it every time it's copied from the reference repo (already stripped from the current template; only relevant if copying fresh content from `~/dev/Class-backstage-software-templates`).
- **The flagged-for-rotation PAT is deliberately live in `integrations.github.token`** (gitignored file, not committed) — don't flag as a new leak, don't suggest rotating ahead of the user's own planned sweep.
- **`rm -rf` on host paths gets denied by a permission hook** for this session/user — use `git rm` for tracked files instead, or ask the user. (Plain `rm` on a single untracked file, as done this session for `test.txt`, is fine.)
- **Don't race the user's interactive `yarn start` shell** with a competing `docker exec`/`yarn start` — EADDRINUSE.
- **`WebSearch` 400s on this backend** — use `ask-codex "web search: ..."` if research is needed.
- **This repo's convention: commit and push `docs/plans/*-RESUME.md` kits directly** — the user has explicitly authorized this for `DevOps2PlatEng1` specifically, overriding the general CLAUDE.md guardrail that normally requires asking first.
