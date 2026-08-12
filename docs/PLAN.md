# PLAN.md

Phased checklist for the Lodgify take-home. Work **one phase at a time**, in
order. Tick boxes as acceptance criteria are met. Do not start a later phase
while an earlier one is unfinished — the phases are ordered so that a stop at
any point still leaves a coherent submission.

Budget: 3–4 focused hours on the day, plus a scaffold built the night before.
If a phase overruns, cut from §Bonus, never from Phases 2–6.

---

## Phase 0 — Scaffold (night before, no brief yet)

Everything here is schema-agnostic and survives whatever the brief says.

- [ ] `src/` package skeleton with `schema.py`, `adapters/`, `clients.py`,
      `prompts/`, `generation.py`, `evaluators/`, `runlog.py`, `analysis.py`
- [ ] `LLMClient` protocol + live Anthropic impl + `StubClient` returning
      canned responses
- [ ] `src/prompts/`: `Prompt` object (id, version, template, `render()` with
      explicit kwargs), one module per family, single `REGISTRY` lookup
- [ ] Run log: append-only, one row per call, all columns from AGENTS.md §3
- [ ] Judge cache keyed on full judge input hash
- [ ] `src/schema.py`: `PropertyListing`, the canonical model, deliberately
      **rich** (optionals, nested amenities, free-text). Prompts and evaluators
      are written against this and do not move tomorrow.
- [ ] `src/adapters/placeholder.py`: an invented raw input shape plus
      `to_listing()`. Exists so there is something to adapt *from* tonight;
      its raw type is imported nowhere else. Kept for tests after the real
      adapter lands.
- [ ] `tests/test_structure.py` with the six structural tests (AGENTS.md §4)
- [ ] PreToolUse hook denying writes to `data/gold_labels.jsonl`
- [ ] `CLAUDE.md` importing `@AGENTS.md`; confirm via `/context`

**Acceptance:** `pytest` green with zero network access. A generation call runs
end to end against `StubClient` and produces one run-log row.

---

## Phase 1 — Adapt to the real brief (~20 min)

- [ ] Read the brief. Note anything contradicting AGENTS.md; flag, don't silently resolve
- [ ] Add `src/adapters/lodgify.py`: their raw shape → the unchanged
      `PropertyListing`. Nothing downstream should need editing. If it does,
      that is a signal the boundary leaked — fix the adapter, not the callers
- [ ] Build the dataset: ~15–20 listings, mixing realistic cases with the
      adversarial slice (AGENTS.md §7)
- [ ] Record in the notebook: what the brief left ambiguous, what was assumed

**Acceptance:** structural tests still pass — their raw type is imported only
inside `src/adapters/`. Dataset loads and every case validates. `git diff`
outside `src/adapters/` and the dataset is empty or near-empty.

---

## Phase 2 — Baseline generator v0 (~20 min)

- [ ] `gen_v0`: deliberately mediocre. No grounding instructions, no length
      guidance, no anti-hallucination framing
- [ ] Generate `n` samples per case, all logged

**Acceptance:** v0 outputs exist for every case with `sample_index` populated.
Do not improve the prompt here — v0's failures are the evidence that later
versions work.

---

## Phase 3 — Deterministic evaluators (~30 min)

Cheap, exact, run before any judge.

- [ ] Format: length bounds, required sections, structured-output validity
- [ ] Placeholder leakage (`[INSERT CITY]`, unfilled templates)
- [ ] Banned phrasing + unverifiable superlatives
- [ ] Fair Housing-style discriminatory language
- [ ] Coverage: were high-value input fields used?

**Acceptance:** every evaluator implements the `Evaluator` protocol, has a unit
test against fixed strings, and returns per-case scores into one DataFrame.

---

## Phase 4 — Grounding judge (~45 min)

The headline metric. Budget the most time here.

- [ ] Claim extraction: generated copy → list of atomic factual claims
- [ ] Claim verification: each claim → supported / contradicted / unsupported
      against the structured input
- [ ] Aggregate to precision and recall over claims
- [ ] Judge calls cached on full-input hash; `judge_version` on every row

**Acceptance:** on a hand-checked case with a known hallucinated amenity, the
judge flags that claim as unsupported. Run the same input twice — cache hit,
identical result, no second API call.

---

## Phase 5 — Judge calibration (~30 min, HUMAN WORK)

- [ ] `TODO(human)`: hand-label ~20 generations. **Not agent-generated** —
      see AGENTS.md §5. This is the point of the section
- [ ] Judge vs hand labels → agreement, and where it disagrees (bias)
- [ ] Judge vs itself, 5 uncached runs at temp 0 → self-consistency (variance)
- [ ] Write the paragraph on what the judge is and isn't trustworthy for

**Acceptance:** both numbers reported side by side, with a plain statement of
what remains unmeasured. After this, judge caching is justified in writing.

---

## Phase 6 — Iterate v1 → v2 (~45 min)

- [ ] `gen_v1`: fix the dominant failure mode v0 showed
- [ ] `gen_v2`: fix the next one
- [ ] Results table: metric per version, with confidence intervals
- [ ] Paired significance test v0→v1 and v1→v2

**Acceptance:** each version change names the failure it targets and shows
whether the improvement survives the significance test. A change that didn't
help stays in the table, reported honestly — that is stronger evidence of
evaluation discipline than three clean wins.

---

## Phase 7 — Adversarial and injection (~20 min)

- [ ] Report scores broken out on the adversarial slice vs the realistic slice
- [ ] Prompt-injection cases: does owner-supplied text steer the output?
- [ ] Name any mitigation applied, and what it costs

**Acceptance:** injection outcome stated explicitly, including if the system
fails. An unmitigated, clearly-reported vulnerability beats a silent one.

---

## Phase 8 — Cost, latency, charts (~20 min)

- [ ] Cost per listing; p50/p95 latency; projection to production volume
- [ ] Score distribution per version
- [ ] Failure-mode breakdown
- [ ] Cost/quality frontier if more than one model tier was tried

**Acceptance:** every chart is generated from the run log, not hand-entered.

---

## Phase 9 — Narrative (~30 min)

- [ ] Assumptions and scope section at the top
- [ ] Each metric justified in customer terms before it is defined
- [ ] Trade-offs, risks, production concerns at the end: judge limits, what
      breaks at scale, online metrics (owner edit-rate on generated copy)
- [ ] Explicit "what I did not build and why" — RAG, fine-tuning, agents,
      observability
- [ ] Read the notebook top to bottom once as a reviewer would

**Acceptance:** a reader who skips all code understands what was measured, how
much to trust it, and what would be done next.

---

## Bonus (only if Phases 1–9 are complete)

- [ ] Second model tier compared on the same eval suite
- [ ] DuckDB query over the results frame
- [ ] Richer statistical treatment (bootstrap CIs, effect sizes)
- [ ] Additional adversarial categories

## Explicitly cut

Named here so the write-up can say they were scoped out on purpose: RAG and
retrieval evaluation, fine-tuning, agent frameworks, chatbot layer, production
observability, real-time serving.

