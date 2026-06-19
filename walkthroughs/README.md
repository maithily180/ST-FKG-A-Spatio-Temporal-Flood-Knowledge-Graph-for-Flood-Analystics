# Query Classification Walkthroughs

This folder traces, step by step, how ST-FKG turns a natural-language question into a
classified query template — and how a **confidence score** is attached to that decision. It
exists to make the classification pipeline *conceptually* understandable: not "the system
picks a template," but exactly *which* rule fires (or why no rule fires and the LLM is called),
*why*, and *where the confidence number comes from*.

Two questions are walked through, deliberately chosen because they are **semantically
identical** but take **two different paths** through the pipeline:

| File | Question | Path | Outcome |
| --- | --- | --- | --- |
| [`example-rules-based.md`](example-rules-based.md) | *"How long was Building_001 flooded?"* | Deterministic rule | `T-03`, confidence **0.99**, no LLM call |
| [`example-llm-paraphrase.md`](example-llm-paraphrase.md) | *"What period of inundation was sustained by Building_001?"* | LLM fallback | `T-03`, confidence **≈0.95** (model self-reported) |

Both resolve to the **same template (`T-03`, flood duration)** and therefore produce the
**same SPARQL and the same answer** — which is the whole point of the design: the meaning is
identical, so the result must be identical, regardless of which path classified it.

## The two-stage classification pipeline

Every question entering `execute_nl_query()` (in `stfkg_query_executor.py`) is classified in a
fixed order:

```
                         ┌─────────────────────────────┐
   NL question  ───────► │  _rule_based_classify()      │   deterministic, no model call
                         │  ~28 ordered if-checks       │
                         └──────────────┬──────────────┘
                                        │
                    rule matched? ──Yes─► source = "rules", use its hardcoded confidence
                                        │
                                        No
                                        ▼
                         ┌─────────────────────────────┐
                         │  classify_query()  (Ollama)  │   LLM classifies intent + params
                         └──────────────┬──────────────┘
                                        │
                                        ▼
                    source = "llm", use the model's self-reported confidence
                                        │
                                        ▼
                    confidence < 0.75 ?  ──Yes─► _fallback_classify() (keyword rules again)
                                        │
                                        No
                                        ▼
                    required params present?  ──No──► ask the user a clarification question
                                        │
                                        Yes
                                        ▼
                    build SPARQL from the template  ──►  execute against AllegroGraph
```

The rule layer runs **first** and acts as a guardrail for common, high-signal phrasings the
LLM is known to confuse. The LLM is only invoked when **no** rule matches.

## How confidence is assigned — two completely different mechanisms

This is the part most people get wrong, so it is stated plainly:

- **Rule path → a hardcoded constant.** Every rule in `_rule_based_classify` returns a
  confidence value that was *written by hand* when the rule was authored, chosen by how
  unambiguous that trigger phrase is. They are not computed from the question. The full set of
  constants actually used in the code is:

  | Confidence | How many rules use it | Meaning the author assigned |
  | --- | --- | --- |
  | `0.99` | 9 rules | phrase is essentially unambiguous (`"how long"`, `"evacuate"`, `"when ... start"`) |
  | `0.98` | 21 rules | strong, specific phrase |
  | `0.97` | 6 rules | strong but slightly broader |
  | `0.96` | 5 rules | broad ranking/listing phrasings |
  | `0.95` | 2 rules | weaker, threshold-dependent matches |
  | `0.92` | 2 rules | generic "depth"/"list flooded" catch-alls |
  | `0.90` | 1 rule | the weakest **partial** match — inferred from *absence* of other signals |

  No rule ever returns `1.0`. The lowest, `0.90`, is the one true "partial match" in the file
  (it infers duration intent without any duration keyword — see the rules-based walkthrough).

- **LLM path → the model's own self-report.** `classify_query` reads the `confidence` field
  straight out of the model's JSON response (`normalized.get("confidence", 0.0)`, defaulting to
  `0.0` only if the field is missing entirely). The prompt asks the model to "use confidence
  between 0.80 and 0.99," but in every logged benchmark run (`research_ablation_results.json`,
  `llm_paraphrase_stress_report.json`) the model returned a **flat 0.95–0.96 for every single
  query**, never varying with difficulty. So LLM confidence is real (not a stub) but **not
  discriminative** — it should not be read as "how likely this is correct."

## The 0.75 threshold

After classification, `execute_nl_query` checks `if confidence < 0.75:` and, if so, re-runs
keyword fallback classification. Because rule confidences start at 0.90 and the LLM self-reports
~0.95, **this threshold is essentially never tripped in normal operation** — both example
questions below clear it comfortably. It is a safety net for malformed or empty LLM responses
(which default to `0.0`), not a routine branch.

## A note on the `Building_001` identifier

The paper writes `Building_001` as a clean, readable example. The **deployed** unit-ID
extractor (`_extract_all_unit_ids`) matches identifiers of the form
`Building_<10–12 hex digits>` via the regex `\b(Building|RoadSegment|...)_([0-9a-f]{10,12})\b`.
A literal three-digit `Building_001` would therefore *not* be extracted, and the pipeline would
fall through to a clarification prompt asking which unit. To keep every step in these
walkthroughs truthful end-to-end, the traces use the real knowledge-graph entity
**`Building_00111dfad8`** (documented in [`../templates.md`](../templates.md): flooded
`TimeInstant_016` → `TimeInstant_054`, peak depth 31.0 m), which `Building_001` stands in for.
