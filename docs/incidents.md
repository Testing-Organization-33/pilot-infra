# Incidents and rollbacks

The release ledger in `releases/` records promotions only. Nothing there shows that production ran
something other than `main` for an hour, so this file is the only place the real production timeline
exists. One entry per rollback or near-miss, newest last.

Format: **date · what shipped · rung chosen · was §4 clean · execution path · elapsed · what the
runbook got wrong**

---

## 2026-08-27 — rehearsal, not a real incident

Rehearsal 10 of the sandbox pilot. Scenario: DASH-901 shipped a payout bug that deployed green
(`pilot-api` and `pilot-web` both moved; only `pilot-api` carried the defect), plus an expand-only
migration in the same range. Friday 18:40, authors unavailable.

- **§4 clean:** yes — `0004_payout_audit.sql` is `CREATE TABLE`, additive.
- **Execution path:** §5a deploy-by-ref (the pilot's deploy workflow has a `ref` input).
- **Rung / elapsed:** recorded by the person who ran it, not by the skill's author.
- **What the runbook got wrong:** 19 defects found, all now fixed in `SKILL.md`. The load-bearing ones:
  §5's pause step came *after* execution, so following the skill in order left promotion open long
  enough to re-ship the bug; `<TARGET>..main` did not resolve because these repos have no local `main`;
  `git revert -m 1` errors on a non-merge commit; the manifest `result` vocabulary was incomplete, so
  `deploy-failure` looked like "nothing landed"; and the skill never said to check whether a repo's
  deploy workflow actually has a `ref` input before recommending deploy-by-ref.

**Carried out of the rehearsal, unresolved:** the four real repos have no rollback path at all. Their
deploy workflows have no `ref` input, and `main` blocks `non_fast_forward` in all four — so neither
§5a nor §5b is available there. Tracked as a prerequisite, not a nice-to-have.
