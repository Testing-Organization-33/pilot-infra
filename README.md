# pilot-infra

Sandbox rehearsal of the DASH release-promotion orchestrator. Owns promotion for `pilot-api` and
`pilot-web`. Everything here is meant to be lifted into `dash-infra` once the ten rehearsals pass —
the only thing that should need editing is `config/repos.json`.

## The model in four sentences

One button. `Promote` pins each repo's `development` tip at run start and ships exactly those SHAs, so
what deploys is what was checked. `main` only ever fast-forwards, so it is always byte-identical to a
tested `development` commit and its history stays provable. Every run writes a manifest — the release
ledger, the changelog, and the restore-point index in one file.

There is no `prepare` step and no candidate state: candidate lifetime is the length of one run, so
there is nothing to go stale. There is no rollback workflow — see `.claude/skills/release-rollback/`.

## Setup

```bash
# One secret. In dash-infra this becomes a GitHub App token instead.
gh secret set PROMOTE_TOKEN -R Testing-Organization-33/pilot-infra
# Optional; without it every report lands in the job summary only.
gh secret set SLACK_WEBHOOK_URL -R Testing-Organization-33/pilot-infra
```

`PROMOTE_TOKEN` needs `contents: write` on `pilot-api`, `pilot-web` and `pilot-infra`, plus
`actions: read` to poll deploys. A classic PAT with `repo` + `workflow` is fine for a sandbox.

## Running a promotion

Actions → **Promote** → Run workflow.

| Input | Use |
|---|---|
| `repos` | `all`, or a comma-separated subset. A subset marks the manifest `partial: true`. |
| `force` | Overrides the batch circuit breaker. Requires `reason`. |
| `reason` | Recorded in the manifest and posted verbatim. Required with `force`. |
| `maintenance_ack` | `none` fails if the batch contains migrations. `maintenance-on` = maintenance screen is up. `no-maintenance` = asserting the migration is expand-only and safe to ship live. |

## What it does, in order

| Phase | Action | Halts on |
|---|---|---|
| 1 Pin | Resolve `development` tip per repo. Nothing downstream reads a branch name again. | — |
| 2 Preflight | Pause check, ancestry, delta, migration scan, circuit breaker (40 commits) | Broken ancestry, active pause, unforced breaker, migrations with no ack |
| 3 CI gate | Verify the pinned SHA has green required checks | Any check red or never run |
| 4 Maintenance | Verify prod maintenance is on *(stubbed in the pilot)* | Maintenance off |
| 5 Ship | Per repo in order: skip only if shipped **and** deployed green, retry a failed deploy, otherwise fast-forward `main` and wait for the prod deploy | Push rejected, deploy failed, redeploy failed, timeout, inconsistent ancestry |
| 6 Record | Tag repos that moved, prune tags > 90d, commit the manifest | — |
| 7 Report | Job summary + Slack, including no-op runs | — |

Fast-forward is enforced server-side: the ref update is sent with `force=false`, so a non-fast-forward
advance returns 422 instead of rewriting anything.

**A failed run is fixed forward and re-pressed.** Phase 5 treats a repo as done only when `main` is at
the pinned SHA **and** the deploy for that SHA went green - those are two separate facts. A repo whose
deploy failed is re-dispatched on the next press rather than skipped, so the run genuinely resumes
where it stopped. There is no recovery mode to learn.

## Where things live

| Path | What |
|---|---|
| `config/repos.json` | Ship order, CI/deploy workflow names, required checks, migration paths, breaker threshold. **The only file to change when consolidating into dash-infra.** |
| `releases/<yyyy>/<ts>.json` | One manifest per run. Every repo recorded whether it moved or not. |
| `state/pause.json` | Expiring promotion pause. Timestamp, never a boolean. 48h cap. |
| `.claude/skills/release-rollback/` | The rollback runbook. |

## Reading a manifest

```bash
ls releases/*/                       # newest last
jq . releases/2026/2026-08-26-1400.json
```

`repos[].from` / `.to` are that repo's `main` before and after. `delta: 0` with `result: "no-delta"` means
it did not move — and it is still recorded, which is what makes every manifest a complete four-repo
snapshot and what makes tags unnecessary as an index.

## Rehearsals

The pilot exists to run these deliberately, not to wait for them. Arm failures via
`pilot-config.json` in the product repos.

| # | Scenario | How | Expected |
|---|---|---|---|
| 1 | Clean promote | Commit to both `development`s, press | Both ship, manifest + 2 tags, ordered |
| 2 | One repo at delta 0 | Commit to `pilot-api` only, press | api ships, web `no-delta`, manifest records web's unmoved SHA |
| 3 | Nothing to ship | Press twice | Second run all `no-delta`, still reports |
| 4 | Red CI | `fail_ci: "unit-tests"` in pilot-api | Phase 3 halt, `main` untouched |
| 5 | Broken ancestry | Push a commit straight to `pilot-api` `main` | Phase 2 halt naming the sync PR; digest alarms |
| 6 | Deploy fails on repo 2 | `fail_deploy: "health"` in pilot-web | api shipped, web `deploy-failure`, halt; re-press skips api and **retries web's deploy** rather than skipping it |
| 7 | Fails after migration | Add a migration + `fail_deploy: "post-migrate-build"` | Schema advanced, `deployed_sha` not — the unrollbackable state, loudly |
| 8 | Breaker tripped | 41+ commits, or drop `breaker_delta` to 1 | Halt; re-press with force + reason; reason lands in the manifest |
| 9 | Migration present | Add a migration file, press with `maintenance_ack: none` | Halt demanding an explicit ack |
| 10 | Manual rollback | Follow the skill after rehearsal 1 | Target resolves from a manifest; pause set |

**Exit criteria:** all ten behave as designed, and rehearsal 10 is executed by someone who did not
write the skill, without asking questions the skill should have answered.

## Known pilot gaps

- Phase 4 maintenance verification is a stub. `dash-infra` must query prod and require `enabled: true`.
- No Slack webhook is configured by default; reports land in the job summary.
- Nothing observes deployed-vs-`main` beyond `deploy-state/state.json`. In production this wants a
  version field on `/health`.
- Deploy timing here is seconds. Real self-hosted deploys take minutes, so the 15-minute poll timeout
  is untested against realistic durations.
