# Multi-Stage Document Investigator

A three-stage LLM pipeline, extraction → contradiction detection → final
reasoning, run against a deliberately messy incident report, with each
stage passing structured JSON to the next instead of re-feeding the raw
document.

## Files

| File | Purpose |
|---|---|
| `source_document.md` | The messy source: a 10-section warehouse-incident report with 6 intentionally planted contradictions (time, certification status, witness count, guard rail condition, job title, alarm activation), repeated boilerplate, and irrelevant filler (cafeteria menu, parking lot notice, company picnic). |
| `pipeline_prompts.md` | The exact prompt text for all three stages, and what each stage is and isn't allowed to see. |
| `runs/temp_0.0/` | Extraction, contradiction, and final-report JSON at the most deterministic setting. |
| `runs/temp_0.7/` | Same three files at a moderate setting. |
| `runs/temp_1.0/` | Same three files at the most permissive setting. |
| `results_comparison.md` | Side-by-side comparison of all three runs, contradictions caught, hallucinations, rule violations, confidence calibration. |
| `failure_analysis.md` | 4 documented incorrect conclusions, each traced to a specific stage, run, and claim ID, with root cause. |
| `obstacle_log.md` | What got in the way while building this and how it was handled. |

## Headline result

| | Temp 0.0 | Temp 0.7 | Temp 1.0 |
|---|---|---|---|
| Contradictions caught | 6/6 | 5/6 | 3/6 |
| Hallucinated facts | 0 | 0 | 2 |
| Assigned fault to an individual | no | no | **yes** (rule violation) |
| Stated confidence | low | medium | high (miscalibrated) |

The full breakdown and the reasoning behind each number is in
`results_comparison.md`; the *why* behind each specific wrong answer is in
`failure_analysis.md`.

## The core finding

Every documented failure traces back to Stage 1 (extraction), a fact was
either invented, dropped, or merged, even in the one case where Stage 3
itself broke its own explicit rules (assigning fault based on a disputed
fact). That rule-break was triggered by Stage 2 already having
editorialized rather than describing the conflict neutrally. The
practical implication: the highest-value place to add guardrails in a
pipeline like this is right after extraction, before anything downstream
has a chance to build on a bad claim.

## Reproducing / extending this

`pipeline_prompts.md` is written so each stage's prompt can be filled in
programmatically (`{document}`, `{stage1_output_json}`,
`{stage2_output_json}`) and looped over real `temperature` values via a
live API call, see `obstacle_log.md` item 1 for why this run was done
manually rather than through a live API in this environment.
