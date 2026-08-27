---
name: release-rollback
description: Recover production after a bad promotion. Use when a release broke prod, someone asks to "roll back the release", a deploy half-applied (migrations ran but the service did not reload), or you need to find the last-known-good SHA for any repo. Covers the four-rung recovery ladder, resolving targets from release manifests, the schema-safety check that decides whether rollback is even legal, and when to stop and escalate instead.
---

# Release rollback

There is deliberately **no rollback workflow**. Ordinary promotion never force-pushes, so no
non-fast-forward bypass exists on `main` for any actor — including automation. Rollback is a human
action, on purpose, with this runbook in front of you.

Read the whole Decide section before touching anything. The wrong rung costs more than the outage.

## 1. Decide which rung

Fastest first. Do not skip to rung 3 because it feels decisive.

| Rung | Do this | Time | Use when |
|---|---|---|---|
| 1 | **Flip the feature flag off** | seconds | The broken feature is flag-gated. No deploy, no git. Nothing here beats it. **Check this first, every time.** |
| 2 | **Fix forward** | ~20-40 min | Default for everything else. PR → `development` → press Promote. |
| 3 | **Manual rollback** (this document, §3-§6) | ~5-10 min | Fix-forward is slower or riskier than reverting, and §4 says rollback is legal. |
| 4 | **Revert PRs** | hours | The bad range must leave `development` too, or the next Promote press re-ships it. Always follows rung 3. |

If the feature was shipped behind a `FeatureKey` flag, you are done at rung 1. Turn it off, then fix
forward calmly. Do not roll back code to disable something a flag already controls.

## 2. First, read what production actually is

Two facts, and they can disagree:

```bash
# What main claims shipped
gh api repos/<ORG>/<REPO>/git/ref/heads/main --jq .object.sha

# What the deploy actually left running
gh api "repos/<ORG>/<REPO>/contents/state.json?ref=deploy-state" --jq .content | base64 -d
```

`state.json` fields: `deployed_sha` (code actually serving), `schema_version` (migrations actually
applied), `last_result`.

**If `schema_version` advanced but `deployed_sha` reads `not-advanced`, stop and read §7.** That is the
half-applied deploy, and rolling code back does not fix it.

## 3. Resolve the target SHA from a manifest

Restore points are addressed **by run timestamp, never by tag**. Tags exist only on repos that moved,
so a repo that sat still during the last release has no tag for it — the manifest is the complete index.

```bash
ls releases/*/            # newest last
cat releases/2026/2026-08-26-1400.json
```

Each manifest records **every** repo, moved or not:

```json
{ "timestamp": "2026-08-26-1400",
  "repos": [ { "name": "pilot-api", "from": "<sha>", "to": "<sha>", "delta": 12, "result": "shipped" },
             { "name": "pilot-web", "from": "<sha>", "to": "<sha>", "delta": 0,  "result": "no-op" } ] }
```

- The **target** is the `to` SHA from the last manifest before the bad one.
- A repo whose `delta` was `0` in the bad run **is not rolled back**. It never moved. Leave it alone.
- `result: "not-attempted"` means the run aborted before that repo — also nothing to roll back.

## 4. Schema check — decides whether rollback is legal

```bash
git -C <repo> fetch --all
git -C <repo> diff --name-only <TARGET>..main -- migrations/
```

| Result | Meaning |
|---|---|
| Empty | Code-only range. Rollback is safe. Proceed. |
| Files, all additive (`CREATE TABLE`, `ADD COLUMN`, new index) | Expand-only. Old code ignores new columns. Rollback is safe. Proceed. |
| Any `DROP`, `RENAME`, `NOT NULL` on an existing column, or a type narrowing | **DO NOT ROLL BACK.** Old code will hit a schema it cannot read, and the migration cannot be undone. Fix forward, and escalate (§7). |

Open the migration files and read them. Do not infer from filenames.

## 5. Execute

**Reverse ship order** — the mirror of promotion. Consumers first, providers last, so nothing is ever
pointed at a dependency that has already moved out from under it.

For the pilot: `pilot-web` → `pilot-api`. For dash-infra: FE → admin → BE.

Per repo, with `<TARGET>` from §3:

```bash
# Preferred: redeploy old code, do NOT rewrite history.
# main keeps pointing at the bad commit; production runs the target.
gh workflow run deploy.yml -R <ORG>/<REPO> -f ref=<TARGET>
gh run watch -R <ORG>/<REPO>
```

This is the whole reason the deploy workflow takes a `ref` input. `main` is untouched, so nothing is
rewritten and no protection is bypassed.

Only if deploy-by-ref is unavailable for that repo, and with a second person watching:

```bash
git -C <repo> fetch origin
git -C <repo> push --force-with-lease=main:<CURRENT_MAIN_SHA> origin <TARGET>:main
```

`--force-with-lease` is mandatory — it refuses if `main` moved since you read it. Never `--force`.
This requires a human with rights; automation cannot do it.

Verify per repo before moving to the next: `state.json` shows the target as `deployed_sha`,
`last_result` is `success`, and the service answers.

## 6. Contain, then land it properly

**Immediately after the last repo:**

```bash
# Without this, the next Promote press re-ships exactly what you just removed.
jq --arg u "$(date -u -d '+48 hours' +%FT%TZ)" \
   --arg r "rolled back <TS> - reverts not yet on development" \
   '.paused_until=$u | .reason=$r' state/pause.json > /tmp/p && mv /tmp/p state/pause.json
git add state/pause.json && git commit -m "pause: rolled back <TS>" && git push
```

48 hours is a cap, not a suggestion. A pause with no expiry becomes a freeze — June 2026 targeted one
week and ran three-plus.

**Then rung 4, same day:** the bad commits are still on `development`. Open revert PRs (`git revert -m 1`
per feature merge commit in `<TARGET>..main`), merge them to `development`, then press Promote normally.
Until that lands, the rollback is temporary by construction.

## 7. Stop and escalate

Wake someone rather than improvising if any of these hold:

- **Destructive migration in range** (§4). There is no rollback. Forward-fix is the only path.
- **Half-applied deploy**: `schema_version` advanced, `deployed_sha` did not. Production is running old
  code against a new schema. Rolling code "back" changes nothing — it is already back. The question is
  whether the new schema is compatible, and if it is not, the fix is forward.
- **Cross-repo contract break**: the consumer shipped expecting a provider version that is no longer
  there. Rolling back one side alone breaks the other. Both move, or neither.
- **`--force-with-lease` refused**: `main` moved while you were working. Someone else is acting. Stop
  and find them before you do anything else.
- **Two rollbacks in one week.** That is a signal about the pipeline, not the release. Raise it.

## 8. Afterwards

Add one line to the incident notes: what shipped, which rung was used, whether §4 was clean, and how
long it took. Two entries in a quarter is the evidence for building a rollback workflow. Zero entries
is the evidence for not having built one.
