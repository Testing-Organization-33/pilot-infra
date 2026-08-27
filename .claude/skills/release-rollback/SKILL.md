---
name: release-rollback
description: Recover production after a bad promotion. Use when a release broke prod, someone asks to "roll back the release", a deploy half-applied (migrations ran but the service did not reload), or you need to find the last-known-good SHA for any repo. Covers the four-rung recovery ladder, locating the defect, resolving targets from release manifests, the schema-safety check that decides whether rollback is even legal, which of the two execution paths this repo actually supports, and when to stop and escalate.
---

# Release rollback

There is deliberately **no rollback workflow**. Ordinary promotion never force-pushes, so no
non-fast-forward bypass exists on `main`. Rollback is a human action, on purpose, with this runbook in
front of you.

Read §1 through §4 before touching anything. The wrong rung costs more than the outage.

## 0. Which execution path does this repo have?

Answer this first, because §5 has two paths and most repos support only one.

```bash
gh api "repos/<ORG>/<REPO>/contents/.github/workflows/deploy.yml" --jq .content | base64 -d | grep -A3 workflow_dispatch
```

| The deploy workflow has… | Your path | Notes |
|---|---|---|
| a `ref` input | **§5a deploy-by-ref** | Preferred. Nothing is rewritten. |
| no `ref` input | **§5b force-with-lease** | Requires a non-fast-forward bypass on `main`, which by design does not exist. If the ruleset blocks you, you have no rollback — go to §7. |

Check per repo, not once. They can differ.

## 1. Decide which rung

Fastest first. Do not skip to rung 3 because it feels decisive.

| Rung | Do this | Time | Use when |
|---|---|---|---|
| 1 | **Flip the feature flag off** | seconds | The broken feature is flag-gated. No deploy, no git. Nothing here beats it. **Check this first, every time.** |
| 2 | **Fix forward** | ~20-40 min | Default for everything else. PR → `development` → press Promote. |
| 3 | **Roll back** (§5) | ~5-10 min | Fix-forward is slower or riskier than reverting, and §4 says rollback is legal. |
| 4 | **Revert PRs** | hours | The bad commits are still on `development`. Until the revert lands, the rollback is temporary — §6. |

If the feature was shipped behind a flag, you are done at rung 1. Turn it off, then fix forward calmly.
Do not roll back code to disable something a flag already controls.

**Flags live in the service's own flag registry.** A CI or deploy toggle file in the repo is not a
feature flag — changing it requires a deploy, which makes it rung 2, not rung 1.

## 2. Locate the defect, then read what production actually is

**First, find out what shipped.** Open the bad run's manifest and read the range:

```bash
cat releases/<yyyy>/<BAD_TS>.json
git -C <repo> fetch --all
git -C <repo> log --oneline <FROM>..<TO>     # FROM/TO from that repo's manifest entry
git -C <repo> show <SHA>                      # read the suspect commit
```

You are looking for which repo carries the defect. Do not roll back a repo just because it appears in
the manifest.

> **Caveat:** a manifest's `from` should equal the previous manifest's `to` for that repo. If they
> disagree, something moved `main` outside the pipeline — a hotfix, or a manual push. Trust `from`
> (it was read at run start) and treat the gap as a separate thing to explain.

**Then read production.** Two facts, and they can disagree:

```bash
# What main claims shipped
gh api repos/<ORG>/<REPO>/git/ref/heads/main --jq .object.sha

# What the deploy actually left running
gh api "repos/<ORG>/<REPO>/contents/state.json?ref=deploy-state" --jq .content | base64 -d
```

`state.json`: `deployed_sha` (code actually serving), `schema_version` (migrations applied),
`last_result`.

**If `schema_version` advanced but `deployed_sha` reads `not-advanced`, stop and read §7.** That is the
half-applied deploy, and rolling code back does not fix it.

## 3. Resolve the target SHA from a manifest

Restore points are addressed **by run timestamp, never by tag**. Tags exist only on repos that moved,
so a repo that sat still during a release has no tag for it — the manifest is the complete index.

```bash
ls releases/*/            # newest last
cat releases/2026/2026-08-27-1355.json
```

Each manifest records **every** repo, moved or not:

```json
{ "timestamp": "2026-08-27-1355",
  "repos": [ { "name": "pilot-api", "from": "<sha>", "to": "<sha>", "delta": 12, "result": "shipped" },
             { "name": "pilot-web", "from": "<sha>", "to": "<sha>", "delta": 0,  "result": "no-delta" } ] }
```

The **target** is the `to` SHA from the last manifest before the bad one, for the repo you are rolling
back. Read `result` to know whether that repo moved at all:

| `result` | Did production move? | Rollback action |
|---|---|---|
| `shipped` | Yes | This is the only value that moved production. Roll back if implicated. |
| `no-delta` | No — nothing to ship | Nothing to roll back. Leave it. |
| `already-shipped` | No — an earlier press landed it | Nothing to roll back in this run. |
| `not-attempted` | No — the run aborted before it | Nothing to roll back. |
| `deploy-failure` / `redeploy-*` | **`main` moved, production may not have** | Read that run's `state.json` before assuming nothing landed. Often §7, not §5. |

**A repo that moved but is not implicated is not rolled back either.** Two repos shipping in one run
does not mean two repos need reverting — roll back the one carrying the defect. §7's contract check
decides whether an uninvolved repo is forced to move with it.

## 4. Schema check — decides whether rollback is legal

```bash
git -C <repo> fetch --all
git -C <repo> diff --name-only <TARGET>..origin/main -- migrations/
```

Use `origin/main`. These repos are checked out on `development`; there is usually no local `main`.

| Result | Meaning |
|---|---|
| Empty | Code-only range. Rollback is safe. Proceed. |
| Files, all additive (`CREATE TABLE`, `ADD COLUMN`, new index) | Expand-only. Old code ignores new columns. Rollback is safe. Proceed. |
| Any `DROP`, `RENAME`, `NOT NULL` on an existing column, or a type narrowing | **DO NOT ROLL BACK.** Old code will hit a schema it cannot read, and the migration cannot be undone. Fix forward, and escalate (§7). |

Open the migration files and read them. Do not infer from filenames.

## 5. Execute

### Pause promotion FIRST

Before you move anything. If you roll back and leave promotion open, the next press re-ships exactly
what you just removed.

```bash
jq --arg u "$(date -u -d '+48 hours' +%FT%TZ)" \
   --arg r "rolled back <BAD_TS> - reverts not yet on development" \
   '.paused_until=$u | .reason=$r' state/pause.json > /tmp/p && mv /tmp/p state/pause.json
git add state/pause.json && git commit -m "pause: rolled back <BAD_TS>" && git push
```

48 hours is a cap, not a suggestion — a pause with no expiry becomes a freeze, which is what June 2026
proved. **Check the calendar**: 48h across a long weekend expires while nobody is watching. Shorten it,
or make sure someone owns Tuesday morning. §6 is what ends the pause; it does not expire your problem.

**`force: true` does not bypass a pause.** Leaving the pause set and then trying to force a promotion
deadlocks — lift it deliberately in §6.

### Order

**Reverse ship order** — the mirror of promotion. Consumers first, providers last, so nothing is ever
pointed at a dependency that has already moved out from under it.

Pilot: `pilot-web` → `pilot-api`. Real repos: FE → admin → BE.

### §5a — deploy-by-ref (preferred, where §0 says it exists)

`main` keeps pointing at the bad commit; production runs the target. Nothing is rewritten and no
protection is bypassed.

```bash
gh workflow run deploy.yml -R <ORG>/<REPO> --ref main -f ref=<TARGET>
sleep 8
RUN=$(gh run list -R <ORG>/<REPO> --workflow deploy.yml --limit 1 --json databaseId --jq '.[0].databaseId')
gh run watch "$RUN" -R <ORG>/<REPO> --exit-status
```

Capture the run id first. `gh run watch` with no id does not pick your run.

### §5b — force-with-lease (only where there is no `ref` input)

This rewrites `main`. It needs a non-fast-forward bypass that by design does not exist — if the ruleset
refuses you, you do not have a rollback. Go to §7 rather than looking for a way around it.

Second person watching, always:

```bash
git -C <repo> fetch origin
git -C <repo> push --force-with-lease=main:<CURRENT_MAIN_SHA> origin <TARGET>:main
```

`--force-with-lease` is mandatory — it refuses if `main` moved since you read it. Never `--force`.

### Verify before moving to the next repo

`state.json` shows the target as `deployed_sha`, `last_result` is `success`, the service answers.

**Expect `schema_version` to name the target's tree even though the newer migration is still applied.**
The marker follows the deployed code, not the database. The extra table is still there — that is fine
if §4 called it additive, and it means the half-applied detector in §7 under-reports until the revert
ships.

## 6. Land it properly

The rollback is temporary until the bad commits leave `development`. Same day:

1. **Find the merges and revert them.**
   ```bash
   git -C <repo> log --merges --oneline <TARGET>..origin/main
   git -C <repo> revert -m 1 <MERGE_SHA>     # for a merge commit
   git -C <repo> revert <SHA>                # for a direct commit - -m 1 errors on a non-merge
   ```
   **Do not let the revert delete an applied migration.** Revert the code; keep the migration file if
   §4 called it additive. Deleting the file does not drop the table, and it makes the next migration
   run inconsistent with the database.
2. **Merge the revert to `development`** through a normal PR.
3. **Lift the pause** — set `paused_until` to `null` and commit. Promotion cannot run while it is set,
   and `force` will not get past it.
4. **Press Promote.** This ships the revert and puts `main` back in agreement with production.
5. *(deploy-by-ref only)* If a repo is stranded on an older ref, that press moves it forward again.
   With §5b the force-push already moved `main`, so there is nothing stranded.

**Nothing in `releases/` records a rollback.** The ledger only records promotions, so a reader
reconstructing history from manifests will not see that production ran something else for an hour.
That is why §8 exists — record the real production state there.

## 7. Stop and escalate

Wake someone rather than improvising if any of these hold:

- **Destructive migration in range** (§4). There is no rollback. Forward-fix is the only path.
- **No execution path** (§0): the deploy workflow has no `ref` input *and* the ruleset blocks a
  non-fast-forward push. Do not go hunting for a bypass mid-incident. Fix forward, and raise that the
  repo has no rollback capability as a separate item.
- **Half-applied deploy**: `schema_version` advanced, `deployed_sha` did not. Production is running old
  code against a new schema. Rolling code "back" changes nothing — it is already back. The question is
  whether the new schema is compatible, and if it is not, the fix is forward. Note that after any
  rollback `schema_version` under-reports, so this detector is unreliable until the revert ships.
- **Cross-repo contract break**: the consumer shipped expecting a provider version that is no longer
  there. Rolling back one side alone breaks the other. Both move, or neither. This is the check that
  decides whether an uninvolved repo has to move.
- **`main` or `development` moved while you were working** — `--force-with-lease` refused, or a push
  was rejected as non-fast-forward. Someone else is acting. Stop and find them first.
- **Two rollbacks in one week.** That is a signal about the pipeline, not the release. Raise it.

## 8. Afterwards

Append one entry to `docs/incidents.md` in this repo. The manifests do not record rollbacks, so this
file is the only place the real production timeline exists.

Format: date · what shipped · which rung · whether §4 was clean · execution path used · elapsed ·
what the skill got wrong.

Two entries in a quarter is the evidence for building a rollback workflow. Zero entries is the evidence
for not having built one.
