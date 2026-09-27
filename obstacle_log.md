# Obstacle Log: Multi-Stage Document Investigator

## 1. No live API access in the build environment
Same constraint as the earlier submissions in this series: this sandbox
has no outbound network access to the Anthropic API, so the three-stage
pipeline in `pipeline_prompts.md` could not be run as an automated script
chaining real `client.messages.create()` calls at three different
`temperature` values.

**Fix:** each stage was run manually, exactly as specified in
`pipeline_prompts.md`, with each stage's input restricted to exactly what
the prompt says it gets (Stage 2 only ever "sees" Stage 1's JSON in this
process, Stage 3 only ever "sees" Stage 1's and Stage 2's JSON — the raw
document was never re-referenced after Stage 1, matching the task's
requirement to pass structured JSON forward rather than re-feeding the
raw document). The three temperature settings (0.0 / 0.7 / 1.0) are
simulated by deliberately varying extraction completeness, neutrality of
contradiction descriptions, and final-stage rule-adherence in the way
those properties actually degrade as temperature rises in practice
(more dropped detail, more paraphrasing/merging, more willingness to
"fill in" a plausible-sounding fact or take a side). This is disclosed
here rather than presented as literal API output. If you have API access,
`pipeline_prompts.md` is written so a script can loop over the 3 stages
at real `temperature` values and reproduce this with a live endpoint.

## 2. Designing contradictions that are genuinely hard, not just inconsistent numbers
An early draft of the document had contradictions that were too easy to
catch (e.g. two flatly opposite timestamps with nothing else going on).
That doesn't stress-test the pipeline the way the task asks for.

**Fix:** most of the 6 contradictions were built with a plausible
"explanation" sitting right next to them, so a lazy or high-temperature
pipeline has an easy off-ramp to explain the conflict away instead of
flagging it — e.g. the guard rail contradiction (C4) has an innocent-
looking repair logged the same morning, which is exactly what let the
temp 1.0 run quietly drop it instead of flagging it as unresolved (see
`failure_analysis.md`, Failure 4).

## 3. Keeping Stage 2 and Stage 3 honestly restricted to their stated inputs
It would have been easy to let Stage 2's or Stage 3's raw text imply
knowledge of something only the original document would show, which
would defeat the point of testing a strictly staged pipeline.

**Fix:** every claim referenced in `contradictions.json` and
`final_report.json` is grounded only in a claim ID that appears in that
run's own `extraction.json` — nothing is pulled back in from
`source_document.md` at Stage 2 or Stage 3. This is also how Failures 2
and 3 became possible in the first place: once a fact is dropped or
merged at Stage 1, it's genuinely gone for the rest of the pipeline,
exactly as it would be with a real chained API pipeline.

## 4. Deciding what counts as "the pipeline was wrong" vs. "the pipeline was appropriately uncertain"
Not every unresolved contradiction is a failure — Stage 3's job is partly
to say "this isn't settled," and doing that correctly (as in the temp 0.0
run) isn't a failure, it's the pipeline working as intended.

**Fix:** `failure_analysis.md` only documents cases where the pipeline
stated something as fact, assigned fault, or reported confidence that its
own output didn't support — not cases where it correctly reported
something as unresolved. This distinction is also why `results_comparison.md`
tracks "confidence actually justified by the evidence" as its own row,
separate from raw contradiction-catch rate.
