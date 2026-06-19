# Walkthrough 1 — Rule-Based Classification

> **Question:** *"How long was Building_001 flooded?"*
>
> **Outcome:** template `T-03` (flood duration), intent `flood_duration`, confidence **0.99**,
> classified by a **deterministic rule** — the LLM is never called.

This is the *canonical* phrasing of a duration question. It contains the exact phrase the rule
layer is built to catch, so it resolves instantly and deterministically.

---

## Step 0 — Entry

The question enters `execute_nl_query("How long was Building_001 flooded?")`. The very first
thing this method does is try the deterministic classifier **before** any LLM call:

```python
classification_source = "rules"
classification = self._rule_based_classify(nl_query)
if classification:
    # deterministic rule matched — LLM is skipped entirely
else:
    classification_source = "llm"
    classification = self.classify_query(nl_query)   # <-- not reached in this example
```

---

## Step 1 — Normalization

Inside `_rule_based_classify`, the text is lowercased and whitespace-collapsed:

```python
query_lower = " ".join(nl_query.lower().split())
# -> "how long was building_001 flooded?"
```

---

## Step 2 — Parameter extraction (`_build_extracted_params`)

Before any rule is checked, the deterministic parameter extractors run once and their results
are reused by every rule:

| Parameter | Extractor | Value here |
| --- | --- | --- |
| `UNIT_ID` | `_extract_all_unit_ids` (regex `Prefix_[0-9a-f]{10,12}`) | `Building_00111dfad8` * |
| `TIME_INSTANT` | `_extract_time_instant` | `None` (no time named) |
| `DEPTH_THRESHOLD` | `_extract_depth_threshold` | `None` → default `0.1` applies later |
| `QUERY_SCOPE` | `_extract_query_scope` | `"single"` (a concrete unit is present) |

```python
has_unit = bool(params.get("UNIT_ID"))   # True
has_time = bool(params.get("TIME_INSTANT"))   # False
```

> \* **ID-format reality:** the paper's `Building_001` is shorthand. The extractor only matches
> 10–12 hex-character IDs, so the real entity being traced is `Building_00111dfad8`. (As the
> next step shows, the rule that fires here does **not** actually depend on `has_unit`, so the
> classification result is the same either way — but the *downstream SPARQL* needs a real ID to
> return data.)

---

## Step 3 — Walking the rule cascade

`_rule_based_classify` is an **ordered list of ~28 `if` checks; the first one that matches
wins and returns immediately.** Every check uses `_contains_any(text, [...])`, a plain
case-normalized substring test. Here is what happens, in order, for our query:

| # | Rule (trigger) | Matches? | Why |
| --- | --- | --- | --- |
| 1 | `"full flood story"` / `"flood story"` | ✗ | phrase absent |
| 2 | `"flood situation"` | ✗ | absent |
| … | event / velocity / risk / `"evacuate"` / `"what events"` rules | ✗ | none of those phrases present |
| n | `"when"` + (`"start"`/`"begin"`/…) → `T-01` | ✗ | the word **"when"** is absent |
| n+1 | `"when"` + (`"stop"`/`"end"`/…) → `T-02` | ✗ | **"when"** absent |
| **▶** | **`_contains_any(query_lower, ["how long", "duration"])` → `T-03`** | **✓** | **"how long"** is a literal substring |

The matching rule is:

```python
if self._contains_any(query_lower, ["how long", "duration"]):
    return result("T-03", "flood_duration", 0.99)
```

Two things worth noticing:

1. **This rule fires on the phrase alone.** It does *not* check `has_unit`, `has_time`, or
   anything else — the presence of "how long" is treated as sufficient, unambiguous evidence of
   a duration question. That is exactly why the author assigned it the top confidence tier.
2. Because it returns *immediately*, the later, weaker fallback rule for the same template —

   ```python
   if (has_unit and "flooded" in query_lower and not has_time
           and not self._contains_any(query_lower, ["when","which","how many","safe","depth"])):
       return result("T-03", "flood_duration", 0.90)
   ```

   is **never reached** for this query. (That 0.90 rule is the subject of the partial-match note
   in the index — it would only be the one to fire if the question said something like *"Was
   Building_00111dfad8 flooded?"* with no "how long".)

---

## Step 4 — Confidence assignment

The returned confidence is the **literal constant `0.99` written into the rule**. It was not
computed from the question. The author chose `0.99` because "how long" / "duration" is, in this
domain, essentially impossible to mean anything other than a duration query.

```json
{
  "query_type": "T-03",
  "intent": "flood_duration",
  "extracted_params": { "UNIT_ID": "Building_00111dfad8", "DEPTH_THRESHOLD": null, ... },
  "confidence": 0.99
}
```

---

## Step 5 — Path & threshold decision

Back in `execute_nl_query`:

- `_rule_based_classify` returned a non-`None` dict → **`classification_source = "rules"`**, and
  `classify_query` (the LLM) is **never invoked**.
- The confidence check `if confidence < 0.75:` → `0.99 < 0.75` is **False** → no fallback.

The classification is accepted as-is.

---

## Step 6 — Required-parameter check

`T-03` declares required params `UNIT_ID` and `DEPTH_THRESHOLD`. `UNIT_ID` is present
(`Building_00111dfad8`); `DEPTH_THRESHOLD` was not extracted but has a default of `0.1`, which
is filled by `substitute_params`. No required parameter is missing → the pipeline proceeds to
build the query (it does **not** ask for clarification).

---

## Step 7 — SPARQL generation & execution

The `T-03` template is instantiated with `UNIT_ID = Building_00111dfad8`, `DEPTH_THRESHOLD = 0.1`:

```sparql
PREFIX ex: <http://www.semanticweb.org/maith/ontologies/2026/2/untitled-ontology-2#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>

SELECT (?endTimeNum - ?startTimeNum AS ?duration)
WHERE {
  BIND(ex:Building_00111dfad8 AS ?unit)
  { SELECT (MIN(?time) AS ?startTime) WHERE {
      ?state a ex:FloodUnitState ; ex:ofUnit ?unit ; ex:occursAt ?time ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
      FILTER(xsd:float(?depth) > 0.1) } }
  # ... matching MAX(?time) subquery for ?endTime ...
  # ?startTimeNum / ?endTimeNum are the integer suffixes of the TimeInstant IRIs
}
```

For `Building_00111dfad8` (flooded `TimeInstant_016` → `TimeInstant_054`, per
[`../templates.md`](../templates.md)), this evaluates the duration as the difference between the
last and first flooded timestep numbers: `54 − 16 = 38` timestep-intervals. At the 6-hour
temporal resolution of the simulation, that is `38 × 6 = 228` hours of inundation.

---

## Summary

```
"How long was Building_001 flooded?"
   │
   ├─ normalize ─────────────► "how long was building_001 flooded?"
   ├─ extract params ────────► UNIT_ID = Building_00111dfad8, QUERY_SCOPE = single
   ├─ rule cascade ──────────► first match: ["how long","duration"]  →  T-03 / flood_duration
   ├─ confidence ────────────► 0.99   (hardcoded constant on that rule)
   ├─ path ──────────────────► source = "rules"   (LLM never called)
   ├─ threshold 0.99 ≥ 0.75 ─► accepted, no fallback
   ├─ required params ───────► UNIT_ID present, DEPTH_THRESHOLD default 0.1  →  OK
   └─ build + execute T-03 ──► duration = last − first flooded timestep
```

Contrast this with [`example-llm-paraphrase.md`](example-llm-paraphrase.md), where the same
meaning is phrased so that **no rule matches** and the LLM must classify it instead.
