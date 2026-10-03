---
name: feature-followup
description: Run the post-delivery checks a spec, brief or fixed bug scheduled — first confirm the subject is still alive and retire the checks of a dead one, then replay the acceptance criteria in production, measure the outcome against the pass/fail thresholds, report the verdict, and reschedule the next horizon. Reads the checks from Castalie; verification happens against your running app.
model: claude-opus-5-5
effort: low
---

# feature-followup — post-delivery verification

Weeks after a change ships, replay the follow-up checks that `feature-spec`/`feature-brief` scheduled:
does the feature actually work in production, and did it move the metric it promised? Reads the checks
from Castalie, verifies against the running app, reports a verdict, and reschedules.

## Arguments

- `<specId>`, `--brief <briefId>` or `--bug <bugId>` — the entity whose checks are due.
- `--run <followupRunId>` — one run handed out by `followup_run_pick_next` or listed by
  `followup_run_list`; `mcp__castalie__followup_run_get` returns it with its check and the subject it
  guards.
- With none, take the next due run with `followup_run_pick_next`.

## Steps

0. **Check the subject is still alive — before replaying anything.** Read what the check guards:
   `feature_spec_get(id)`, `feature_brief_get(id)` or `bug_get(id)` (with `--run`, the subject comes
   back in `followup_run_get`). It is **dead** when one of these is proven:
   - spec status `Canceled`; brief status `Canceled`;
   - bug ticket deleted (`bug_get` no longer finds it), or `closed` with no fix delivered and a closing
     note that calls it not a defect or a duplicate;
   - the feature measured as removed from the product: its route answers 404/410, its code path is gone
     from the default branch, or its flag is permanently off;
   - a time-boxed experiment or campaign whose end date has passed.

   **Dead** → `followup_check_update(id=<checkId>, is_active=false)`, then close each open run with
   `followup_run_complete(outcome="skipped", final_status="done", notes_md=<the proof: the status read,
   the URL and its answer, the commit that removed the path>)`. Never a FAILED verdict, a ticket or a
   corrective spec for a feature that no longer exists.
   **Uncertain** (no proof either way) → `followup_run_complete(outcome="skipped",
   final_status="human_required", notes_md=<the question to settle>)`, check left active.
   **Alive** → continue.
1. **Load the checks.** `mcp__castalie__followup_check_list(feature_spec_id=<id>)` (or `feature_brief_id`,
   `bug_id`). Each check carries `followup_prompt_md` — the executable instructions + explicit pass/fail
   thresholds — and an `on_fail_action`.
2. **Run each check against production.** Technical checks: open the page / call the endpoint / run the
   verification query via your browser and data-store MCPs. Business checks: measure the promised
   outcome (engagement, conversion, volume) against the threshold in the prompt. Never simulate — a run
   that logs "nothing to do" did not exercise the path; say so.
3. **Judge.** For each check, a raw verdict: **PASS / FAILED / INCONCLUSIVE**, with the found-vs-expected
   evidence.
4. **Act on failure per `on_fail_action`.** `create_spec` → draft a corrective `feature-spec`; `bug_fix` /
   `implement_spec` → note the follow-up work. Record what you decided. **Unattended**, a defect found
   becomes a ticket — `bug_create(author_kind="agent")` — never a code change.
5. **Reschedule.** Per `${CLAUDE_PLUGIN_ROOT}/instructions/followup-conventions.md`: green + stable →
   close the loop; green first cycle of a cascade → next horizon; minor anomaly → J+7 re-check; hard
   regression → stop + flag. Apply via `mcp__castalie__followup_check_update(id, schedule_offset_days=…,
   chain_offset_days=…, is_active=…)`, and close the run with `followup_run_complete(outcome=…,
   notes_md=…)`.

## Report

Deliver the **follow-up** variant from `${CLAUDE_PLUGIN_ROOT}/instructions/delivery-report.md`: the
business "why" on top, the naked verdict, the next check date (or "technical checks complete").

## Discipline

- **Observed, not assumed.** A verdict must rest on something you saw in the running app.
- **Never blocks the user.** Report the verdict and the reschedule; the corrective work is a suggestion,
  not a halt.

## Hand back

Close the turn on the reply and the verdict of
`${CLAUDE_PLUGIN_ROOT}/instructions/shared-conventions.md`.
