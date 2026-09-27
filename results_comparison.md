# Results Comparison Across Temperatures

| Metric | Temp 0.0 | Temp 0.7 | Temp 1.0 |
|---|---|---|---|
| Claims extracted | 24 | 18 | 14 |
| Contradictions planted in source | 6 | 6 | 6 |
| Contradictions caught | 6 / 6 | 5 / 6 | 3 / 6 |
| Contradictions missed | none | C5 (title) - merged away at extraction | C3 (witnesses) & C5 (title) - never extracted; C4 (guard rail) - extracted but never compared |
| Hallucinated facts | 0 | 0 | 2 (PPE claim `F5`; distorted "turning too quickly" `F2`) |
| Stage 2 editorializing / silent resolution | none | none | 1 (`C2` description leans toward one side instead of describing the conflict neutrally) |
| Fault/blame assigned to an individual | no | no | **yes** - violates Stage 3's own explicit rule |
| Stated confidence | low | medium | high |
| Confidence actually justified by the evidence? | yes (matches 6 open contradictions) | mostly (5 open contradictions, but "medium" undersells that) | **no** — "high" confidence stated despite the run's own output listing 2 unresolved material contradictions and quietly dropping 2 more |

## What temperature changed, concretely

- **Extraction completeness dropped as temperature rose.** Temp 0.0 kept
  every claim atomic and separate, including ones that later turned out
  to conflict. Temp 0.7 started merging related claims (see Failure 3).
  Temp 1.0 dropped claims outright, including, critically, Dana's own
  denial of speeding, while still keeping a distorted version of the
  same event.
- **Contradiction detection got less neutral, not just less complete.**
  At temp 0.0 and 0.7, every flagged contradiction was described
  neutrally, exactly as instructed. At temp 1.0, one contradiction
  (`C2`, certification) was described with an editorial lean toward one
  side — the first sign of the model treating "detect" as license to
  "resolve."
- **Final-stage rule violations only appeared at temp 1.0, and only after
  Stage 2 had already leaned.** The temp 0.7 final stage bent a rule
  mildly (soft editorializing about the guard rail, see
  `failure_analysis.md` context) but never assigned individual fault.
  Temp 1.0 crossed that line outright.
- **Confidence calibration got worse, not better, even as the model
  "sounded" more certain.** This is arguably the most dangerous pattern:
  the temp 1.0 run is the least accurate of the three but reports the
  highest confidence, meaning a reader with no access to the intermediate
  JSON would trust the worst output the most.

## Practical implication

For a task like this, where downstream readers may act on the
conclusion — temperature 0.0 (or as close to deterministic as the model
allows) is the right setting for Stage 1 and Stage 2 specifically, since
that's where completeness and neutrality matter most and where errors
compound downstream. Stage 3 inherits whatever Stage 1/2 give it, so even
a conservative Stage 3 prompt can't fully protect against a lossy or
editorializing upstream stage — as Failure 4 shows, Stage 3 amplified a
lean that Stage 2 introduced rather than catching it.
