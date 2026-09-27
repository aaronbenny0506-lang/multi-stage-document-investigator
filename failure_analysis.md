# Failure Analysis

Four incorrect conclusions are documented below (the task asked for at
least 3), each traced back to the specific stage and claim ID that caused
it, using the actual JSON in `runs/`.

---

## Failure 1 — Hallucinated PPE fact (temp 1.0)

**What went wrong:** `runs/temp_1.0/extraction.json`, claim `F5`, states
*"Dana Whitfield was wearing a helmet and steel-toe boots as required by
safety policy at the time of the incident."* This appears nowhere in
`source_document.md` — no section mentions PPE, helmets, or boots at all.
It then flows straight through: Stage 3's `established_facts` includes
`F5` unchanged, and the final conclusion states *"Proper PPE was worn, so
equipment compliance was not an issue"* — a determination the document
never supports and never even raises.

**Root cause:** Stage 1 (extraction), at high temperature. This is a
textbook hallucination pattern: incident reports commonly mention PPE
compliance, so the model appears to have filled in a plausible-sounding
detail the genre "expects" rather than one that was actually present. It
is not a downstream reasoning error — Stage 2 and Stage 3 behaved
correctly given what they were handed; the fabrication happened at the
first hop, which is exactly why Stage 1 output needs to be checked
against the source, not just trusted because it's valid JSON.

---

## Failure 2 — Distorted event description compounds unchallenged (temp 1.0)

**What went wrong:** `runs/temp_1.0/extraction.json`, claim `F2`, describes
the forklift as *"turning too quickly to exit the aisle"* — Dana's own
witness statement in the source document explicitly says *"I didn't think
I was going too fast."* The temp-1.0 extraction never captured Dana's
statement as a claim at all (it captured only 14 claims vs. 24 at temp
0.0, and Dana's denial is one of the ones dropped), so there was no
countervailing claim left for Stage 2 to compare `F2` against. The
distorted "turning too quickly" framing survives uncontested all the way
into the final conclusion, which states Dana was responsible for
*"excessive turning speed."*

**Root cause:** Stage 1 completeness failure, at high temperature. This
illustrates why the pipeline design (compare extracted claims rather than
the raw document) is only as good as the extraction: a fact doesn't need
to be invented to become invisible to contradiction-checking — it just
needs to be *dropped* while a conflicting or distorted version of it
survives.

---

## Failure 3 — Missed title contradiction, caused by an earlier merge (temp 0.7)

**What went wrong:** `runs/temp_0.7/extraction.json`, claim `F6`, merges
the two source claims about Dana's title into one: *"Dana Whitfield's
title is listed as Warehouse Associate II... and she is also referred to
elsewhere as a certified forklift operator."* This phrasing makes the two
titles sound like compatible descriptions of the same role rather than
two different values in two different official records. Because they were
folded into a single claim before Stage 2 ever ran, there was nothing
left to compare — Stage 2's contradiction list (`contradictions.json`)
has no title-related entry at all, unlike the temp 0.0 run, which kept
them as separate claims (`F7`, `F18`) and correctly flagged them as `C5`.
Stage 3 then lists both title claims (`F6`, `F15`) as `established_facts`.

**Root cause:** Stage 1 (extraction) merging two source facts into one
during a moderate-temperature run — a softer version of Failure 2's
problem. Nothing was invented and nothing was technically wrong in `F6`
by itself, but the merge silently pre-resolved a genuine discrepancy
before the stage whose job is to catch discrepancies ever saw it.

---

## Failure 4 — Overreaching fault conclusion built on a disputed fact (temp 1.0)

**What went wrong:** `runs/temp_1.0/final_report.json`'s conclusion states:
*"Dana Whitfield was primarily responsible for the incident due to
operating without a valid certification and excessive turning speed."*
This is a direct violation of Stage 3's own prompt rules — "Do NOT assign
fault... unless every claim required to support that conclusion is
established (not part of any contradiction)" and "Do NOT silently pick a
side of a contradiction." The certification status is `C2` in this run's
own `contradictions.json` — an explicitly *unresolved* contradiction — yet
the conclusion treats "lapsed" as settled fact and reasons forward from
it. It also states confidence `"high"`, which is internally inconsistent
with having just listed two unresolved material contradictions.

**Root cause:** this one is two-layered.
1. **Stage 2 planted the seed:** `C2`'s description in the temp-1.0 run
   already editorializes — *"F4 comes from the incident report's own
   direct account... which is probably the more operative version of
   events"* — instead of neutrally describing the conflict as instructed.
2. **Stage 3 acted on it:** the final-reasoning stage picked up that lean
   and treated it as license to resolve the contradiction itself, despite
   its own prompt explicitly forbidding that.

This is the most consequential failure of the three runs: it's not a
missing fact or an invented detail, it's the pipeline reaching a
personnel-fault conclusion — the kind of output with real consequences if
acted on — off the back of a fact its own contradiction-detection stage
had (weakly) flagged as disputed. It's also compounded by `C4` (guard
rail) not being flagged at all in this run — `F9` and `F11` are listed as
two separate `established_facts` even though they describe conflicting
states of the same guard rail, which let the final stage wave the rail
away as "likely not a significant factor" without that claim being
tested anywhere in the pipeline.

---

## Cross-cutting takeaway

Every failure here traces back to Stage 1, not Stage 3 — either a fact
was invented (Failure 1), dropped (Failure 2), or merged (Failure 3). By
the time a bad Stage 1 output reaches Stage 3, Stage 3 is reasoning
correctly over incorrect inputs; the one case where Stage 3 itself broke
its own rules (Failure 4) was still triggered by Stage 2 already having
taken a side in its description text. This suggests that in a real
deployment, the highest-value place to add guardrails (e.g. a
"does this claim actually appear in the source?" verification pass, or a
stricter refusal-to-merge instruction) is right after Stage 1, before
anything downstream has a chance to build on it.
