# Walkthrough 2 — LLM-Assisted Classification (Paraphrase)

> **Question:** *"What period of inundation was sustained by Building_001?"*
>
> **Outcome:** template `T-03` (flood duration), intent `flood_duration`, confidence **≈0.95**
> (the model's self-report), classified by the **LLM fallback** because **no deterministic rule
> matches**.

This is a *semantically identical* paraphrase of Walkthrough 1 — it asks for exactly the same
thing (how long a unit was flooded). But it is worded to avoid every trigger phrase the rule
layer knows, so it deliberately exercises the LLM path. The key takeaway: it still lands on the
**same `T-03` template**, so it produces the **same SPARQL and the same answer** — the two paths
converge.

---

## Step 0–2 — Entry, normalization, parameter extraction

Identical to the rules-based walkthrough. `execute_nl_query` calls `_rule_based_classify` first;
the text is normalized to:

```python
query_lower = "what period of inundation was sustained by building_001?"
```

and `_build_extracted_params` extracts:

| Parameter | Value |
| --- | --- |
| `UNIT_ID` | `Building_00111dfad8` * |
| `TIME_INSTANT` | `None` |
| `DEPTH_THRESHOLD` | `None` (default 0.1 later) |
| `QUERY_SCOPE` | `"single"` |

> \* Same `Building_001` → real-ID shorthand as in Walkthrough 1 (see the index's ID-format
> note).

---

## Step 3 — The rule cascade runs… and **nothing matches**

This is the crux. The rule layer is pure substring matching, and this paraphrase was written to
miss every trigger. Walking the relevant checks:

| Rule (trigger phrases) | Matches? | Why it fails |
| --- | --- | --- |
| `["how long", "duration"]` → `T-03` | ✗ | the query says **"period of inundation"**, not "how long" or "duration" |
| `has_unit and "flooded" and not has_time and not [...]` → `T-03` (the 0.90 fallback) | ✗ | requires the literal word **"flooded"** — the query says **"inundation"** / **"sustained"** instead |
| `"when"` + start/end words → `T-01`/`T-02` | ✗ | no **"when"** |
| `["timeline","history","over time","all depths","flood story"]` → `T-04` | ✗ | none present |
| `\bsafe\b` → `S-03` | ✗ | no "safe" |
| `"depth"` + has_unit → `S-02` | ✗ | no "depth" |
| `["which","list","show"]` + flooded → `S-04` | ✗ | none present |
| *(all other ~20 rules)* | ✗ | their phrases are absent |

Every `if` falls through, so the function reaches its final line:

```python
return None
```

A vocabulary mismatch is the entire reason: "period of inundation … sustained" is perfectly
clear to a human, but it shares **no literal substring** with any rule's trigger list. This is
precisely the gap the LLM layer exists to cover.

---

## Step 4 — Escalation to the LLM (`classify_query`)

Because `_rule_based_classify` returned `None`:

```python
classification_source = "llm"
classification = self.classify_query(nl_query)
```

`classify_query` sends the question to the local Ollama model with a compact prompt
(`build_fast_llm_prompt`) that contains the 16 template IDs, a short hint list, and a JSON
schema. The hint list explicitly includes:

```
- how long/duration -> T-03
- timeline/history/flooding situation -> T-04
...
"confidence": 0.85          <-- example value shown in the schema
- Use confidence between 0.80 and 0.99 when you can classify.
```

The model must now do what the rules could not: **infer semantically** that "what period of
inundation was sustained" means *duration*, and map it to `T-03`. Critically, the model's job is
**only to classify intent and extract parameters** — it is *never* asked to write SPARQL. The
returned JSON looks like:

```json
{
  "query_type": "T-03",
  "intent": "flood_duration",
  "extracted_params": { "UNIT_ID": "Building_001", "TIME_INSTANT": null, ... },
  "confidence": 0.95
}
```

---

## Step 5 — Where the confidence comes from (and its caveat)

`classify_query` reads the confidence **directly from the model's JSON**, with a hard default
only if the field is missing:

```python
confidence = normalized.get("confidence", 0.0)
```

So the `0.95` here is the **model's own self-assessment**, not a computed quantity and not a
hardcoded rule constant. The important, honest caveat (verified against the logged benchmark
runs in `research_ablation_results.json` and `llm_paraphrase_stress_report.json`): the model
returns essentially the **same 0.95–0.96 for every question it classifies**, regardless of how
hard or easy the paraphrase is. The prompt invites 0.80–0.99, but in practice the value barely
moves. So for the LLM path, confidence indicates "the model produced a classification," **not**
"this classification is more/less likely to be right."

---

## Step 6 — Parameter normalization (the paraphrase's hidden risk)

`_normalize_llm_classification` post-processes the model's `extracted_params`. It re-runs the
deterministic ID extractor on whatever the model returned for `UNIT_ID`:

```python
unit_candidate = self._extract_building_id(str(raw_unit))
unit_id = unit_candidate or default_unit or raw_unit
```

With a real 10–12 hex ID present in the question, this recovers the canonical
`Building_00111dfad8`. (If the model had echoed a non-existent literal like `Building_001`, that
string would survive to the SPARQL and the query would execute successfully but return **zero
rows** — a failure mode the confidence score cannot see, because confidence only reflects
*template* certainty, never whether the entity actually exists.)

---

## Step 7 — Threshold, build, execute — identical to the canonical path

- `confidence 0.95 < 0.75` → **False** → no keyword fallback.
- `T-03` required params `UNIT_ID` (present) and `DEPTH_THRESHOLD` (default 0.1) → nothing
  missing → no clarification.
- The **same `T-03` template** is instantiated with the **same parameters** as Walkthrough 1,
  so the generated SPARQL is **byte-for-byte identical**, and the executed answer is the same
  duration (`54 − 16 = 38` timestep-intervals → 228 hours for `Building_00111dfad8`).

---

## Summary — and why this matters

```
"What period of inundation was sustained by Building_001?"
   │
   ├─ rule cascade ──────────► NO rule matches ("inundation"/"sustained" ≠ any trigger phrase)
   │                            → _rule_based_classify returns None
   ├─ escalate to LLM ───────► classify_query()  →  T-03 / flood_duration
   ├─ confidence ────────────► 0.95   (model self-reported; flat across all queries)
   ├─ path ──────────────────► source = "llm"
   ├─ normalize params ──────► UNIT_ID recovered to Building_00111dfad8
   ├─ threshold 0.95 ≥ 0.75 ─► accepted, no fallback
   └─ build + execute T-03 ──► SAME SPARQL, SAME answer as the canonical phrasing
```

| | Walkthrough 1 (canonical) | Walkthrough 2 (paraphrase) |
| --- | --- | --- |
| Trigger phrase present? | yes (`"how long"`) | no |
| Classified by | deterministic rule | LLM |
| Confidence | 0.99 (hardcoded constant) | ≈0.95 (model self-report) |
| LLM called? | no | yes |
| Template / SPARQL / answer | `T-03` | **same `T-03`** |

The deterministic rule layer gives the canonical question a fast, reproducible, model-free
answer; the LLM layer makes the system **robust to rewording** by recovering the same intent
when the rules' vocabulary runs out. Either way, classification only ever *selects a template* —
the SPARQL itself is always generated deterministically from that template, which is what keeps
both answers consistent and explainable.
