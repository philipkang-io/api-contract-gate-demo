# API Contract Gate — Demo Runbook

A CI/CD gate demo: a spec-first API where every push and every PR is linted against
governance rules and contract-tested against the running service. The point of the
demo is not that the pipeline goes green — it is that **a contract break cannot reach
`main`**.

- **Repo:** `philipkang-io/api-contract-gate-demo` (private)
- **Local:** `~/Demo/Github/api-contract-gate-demo`
- **Workspace:** `[philip] api-contract-gate-demo` (`dceb00e5-6169-4a7d-9964-73f946662ab8`, team philip V12)
- **Workflow:** `.github/workflows/api-contract-gate.yml`
- **Runtime:** ~1m45s per CI run

---

## One-time setup

Both are UI actions and both are required before the demo works end to end.

### 1. Branch protection on `main`

Settings → Branches → add a rule for `main`:

- ✅ **Require status checks to pass** → select **`Validate API Spec & Contract`**
- ❌ **Do NOT** enable *"Require a pull request before merging"*

Without this, a red PR is still mergeable and Act 2's "the break is blocked" is only
narration. With *require-PR* enabled, Act 1's direct push to `main` is rejected.

### 2. Bind Postman desktop to the local folder

Open the desktop app → workspace `[philip] api-contract-gate-demo` → connect it to
`~/Demo/Github/api-contract-gate-demo`. The collection is then edited as files in the
repo rather than in Postman Cloud.

**Immediately afterwards run `git status`.** This operation previously corrupted a
sibling repo — it deleted the collection folder and wrote a duplicate `<name>-1/`.
Expect no changes, or at most an update to `.postman/resources.yaml`. If a `-1` folder
appears or collection files show as deleted, stop before committing.

Sanity check: desktop shows collection **Book API** with **8 requests**.

---

## Pre-flight (before the call)

| Check | Required state |
|---|---|
| `main` | green, latest run visible |
| Open PRs / demo branches | none — created live |
| Local repo | on `main`, clean, `git pull` done |
| Postman desktop | open on the workspace, 8 requests visible |
| Local server | `npm start` on :3000 — only if running the collection inside the app |
| Browser tabs | ① repo files ② Actions ③ **last successful run, already open** (fallback) |

---

## Act 0 — Set the scene (~1 min)

Show the repo tree: `postman/specs/`, `postman/collections/Book API/` as
`.request.yaml` files, `.github/workflows/`.

> "Your contract lives in your repo, next to your code. None of this is in Postman's cloud."

---

## Act 1 — The loop (~4 min)

In the **desktop app**, add an assertion to any request. Save. Show the `.request.yaml`
changed on disk. Then:

```bash
git commit -am "test: add assertion"
git push
```

**Expected:** lint clean, `8 requests`, **19 assertions** (up from 18). The changed count
is the proof that a desktop edit reached CI.

> ⏱ Start the Act 2 break while this run is still going — it saves ~2 minutes of dead air.

---

## Act 2 — The break (~5 min)

```bash
git checkout -b demo/contract-break
# src/server.js line 14:  status: 'ok'  ->  state: 'ok'
git commit -am "refactor: rename health response field status -> state"
git push -u origin demo/contract-break
gh pr create --fill
```

**Expected — red:**

```
Pass  Status code is 200
1. AssertionError  Response has status ok
   expected undefined to deeply equal 'ok'

requests   | 8  executed | 0 failed
assertions | 18 executed | 1 failed
```

Then show the merge button blocked.

> "The service is up. It returns 200. Health checks pass. And every consumer of this
> endpoint just broke."

---

## Act 3 — The fix (~4 min)

```bash
# revert src/server.js line 14 back to  status: 'ok'
git commit -am "fix: restore health response field 'status'"
git push
```

**Expected:** the *same* PR flips green — `8/8 requests, 18 assertions`. Merge it.

> "`main` never saw the break."

---

## Reset after the demo

```bash
git checkout main && git pull && git branch -D demo/contract-break
```

---

## Fallbacks

| Problem | Do this |
|---|---|
| CI queued or slow | Walk tab ③ (a completed run) while the live one catches up |
| Desktop misbehaves on save | Edit in your editor — the story is the gate, not the client |
| Unrecoverable | Open a prior run in Actions showing the red → green history |

---

## Notes

- The service is **Node**. For a .NET/Java audience, narrate the `npm` steps as
  "your build step goes here" rather than skipping past them.
- **No `--report-events`.** It errors on file-path runs (no cloud collection UUID to
  attribute to) and lint-by-file-path with it spawns phantom governed-spec records
  in the API Catalog. The Catalog is a different demo.
- **The empty-run guard is load-bearing.** A collection run that executes nothing exits
  `0`. Without the guard, a broken collection reports a green gate having tested
  nothing — which is exactly how a sibling repo stayed "green" for two months while
  running zero requests. The guard fails the job on: 0 requests executed, fewer
  executed than `.request.yaml` files on disk, or 0 assertions.
- **Governance rules.** `spec lint --workspace-id` currently resolves to Postman's
  default ruleset because the workspace has no custom rules. Defaults have real teeth
  (a degraded spec produces 23 governance errors), but until custom rules are
  configured, do not claim "your organisation's standards are enforced."
