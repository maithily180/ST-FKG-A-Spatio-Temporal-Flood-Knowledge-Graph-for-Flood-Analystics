# ST-FKG SPARQL Query Template Library

This directory documents every natural-language → SPARQL query pattern implemented in
`stfkg_query_executor.py` (`STFKGQueryExecutor`), organized into the six query families the
project defines:

| Series | File | Family | Framing |
| --- | --- | --- | --- |
| **S** | [`S-series.md`](S-series.md) | State (Spatial) Queries | Infrastructure flood-state retrieval and spatial flood-status analytics |
| **T** | [`T-series.md`](T-series.md) | Temporal Queries | Flood arrival, duration, persistence, and temporal flood-evolution analysis |
| **A** | [`A-series.md`](A-series.md) | Aggregate Queries | Infrastructure ranking and flood-severity aggregation |
| **E** | [`E-series.md`](E-series.md) | Event Queries | Flood-event occurrence and event-history analytics |
| **R** | [`R-series.md`](R-series.md) | Risk Queries | Flood-risk assessment from hazardous velocities and critical-infrastructure depth exposure |
| **D** | [`D-series.md`](D-series.md) | Decision-Support Queries | Operational evacuation and infrastructure-prioritization analytics |

## Two layers of query pattern

The executor implements query patterns at two layers:

1. **16 static templates**: the `TEMPLATES` dict in `stfkg_query_executor.py`:
   `S-01`–`S-04`, `T-01`–`T-04`, `A-01`–`A-03`, `E-01`, `E-03`, `R-01`, `R-03`, `D-01`. These are
   the core templates referenced by the LLM classification prompt
   (`build_llm_prompt`/`build_fast_llm_prompt`), by `REQUIRED_PARAMS`, and by `INTENT_MAPPING`.
2. **Dynamic intent variants**: `build_query()`'s `intent`-keyed dispatcher generates a more
   specific SPARQL body for the same `template_id` when the classifier detects a more specific
   phrasing: a named entity-type scope, a plural/"all units" scope, a comparison between two
   units, a named scenario, and so on. These extend each family's analytical reach beyond the
   base template and are documented per family under an **"Extended dynamic patterns"** section.

Each per-template entry in the family files states which layer it belongs to.

## How to read each entry

Every static template entry contains:

- **Template ID & name**, as declared in `TEMPLATES`.
- **Purpose**: the plain-English description from `TEMPLATES["description"]`, expanded with
  the operational framing from `explanation_builder.TEMPLATE_EXPLANATIONS` (`human_intent`,
  `logic`, and `method` steps, the same text the system uses to explain a result to an end user).
- **Parameters**: required vs. optional, with defaults from `DEFAULT_PARAMS` where they apply.
- **Rule-trigger phrases**: the literal substrings `_rule_based_classify` checks for, so you
  know exactly which wordings are answered deterministically (no LLM call) versus which fall
  through to Ollama classification. See [Substitution & prefixes](#substitution--prefixes) for
  substitution mechanics.
- **SPARQL**: the literal `query` string from `TEMPLATES`, `{PARAM}`-placeholders intact.
- **Worked example(s)**: a real natural-language question, its extracted parameters, the fully
  substituted SPARQL, and the result. Worked examples use real entity IDs and depth/velocity
  values pulled from the generated KG (`output/*.ttl`); see
  [Real KG values used in this library](#real-kg-values-used-in-this-library).

Each "Extended dynamic patterns" subsection documents the intent name, trigger phrase(s), the
generated SPARQL, and how it extends the static template for that `template_id`.

## Substitution & prefixes

Every template's `{PLACEHOLDER}` tokens are substituted by `substitute_params()` before
execution, and every final query is prefixed with:

```sparql
PREFIX ex: <http://www.semanticweb.org/maith/ontologies/2026/2/untitled-ontology-2#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
PREFIX geo: <http://www.opengis.net/ont/geosparql#>
```

(`STFKGQueryExecutor.SPARQL_PREFIXES`). These prefixes are omitted from the per-template SPARQL
blocks below to keep them focused. Prepend them when running a query against AllegroGraph.

## Default parameter values

`DEFAULT_PARAMS` (used whenever a value isn't extracted from the question or supplied by the
caller):

| Parameter | Default | Meaning |
| --- | --- | --- |
| `DEPTH_THRESHOLD` | `0.1` (m) | The depth above which a unit counts as "flooded", the same threshold the simulation uses to materialize a `FloodUnitState` (see `ONTOLOGY.md` §3) |
| `VELOCITY_THRESHOLD` | `2.0` (m/s) | The flow speed above which a unit counts as facing "high-velocity risk" |
| `MIN_DURATION` | `20` (timesteps) | Used by the `high_risk_check` dynamic pattern (A-series) as the minimum flood duration to count as "high risk" |
| `LIMIT_VALUE` | `20` | Default result cap for list/ranking queries |
| `TIME_INSTANT` | `TimeInstant_011` | The reference "current" time instant when a question doesn't name one explicitly |

## Parameter extraction reference

Parameters are filled from explicit caller input where available, and otherwise extracted from
the question text by `_build_extracted_params()`, which runs a battery of extractors used by
both the deterministic rule path and the LLM path:

| Parameter | Extracted by | Recognizes |
| --- | --- | --- |
| `UNIT_ID` / `SECOND_UNIT_ID` | `_extract_all_unit_ids` / `_extract_building_id` | `Building_<hex10-12>`, `RoadSegment_…`, `RailwaySegment_…`, `Hospital_…`, `School_…`, `PowerFacility_…`, or typed phrasing like "road segment 000219a31d" / "hospital 0249c5e672" |
| `TIME_INSTANT` / `SECOND_TIME_INSTANT` | `_extract_all_time_instants` / `_extract_time_instant` | `TimeInstant_011`, `time instant 11`, `timeinstant_11`, `at time 11` → normalized to `TimeInstant_NNN` (zero-padded to 3 digits) |
| `DEPTH_THRESHOLD` | `_extract_depth_threshold` | `depth > 0.5`, `flooding (> 2.0 m`, `unusable (depth >= 1.5`, `impassable (depth > …`, `over 2.0 m`, `above 1.5 m` |
| `VELOCITY_THRESHOLD` | `_extract_velocity_threshold` | `velocity > 2.0`, `> 2.0 m/s` |
| `EVENT_TYPE` | `_extract_event_type` | `<Name> events`, `<Name> occurred` (e.g. "RapidRise events", "PeakFlood occurred") |
| `LIMIT_VALUE` | `_extract_limit_value` | `top 10`, `first 5`, `show me 20`, `limited to 15` |
| `ENTITY_TYPE` | `_extract_entity_type` | A concrete `Prefix_hexid`, or category words: "critical infrastructure" → `CriticalInfrastructure`; "power facility/facilities" → `PowerFacility`; "hospital(s)", "school(s)", "railway segment(s)", "road segment(s)/road(s)", "building(s)" |
| `QUERY_SCOPE` | `_extract_query_scope` / `_has_plural_scope` | `"single"` if a concrete unit ID is present; otherwise `"plural"` if the text contains broad phrasing such as "which buildings", "all units", "each unit", bare plural nouns ("roads", "hospitals", "schools", …), or "critical infrastructure" |
| `SCENARIO_ID` | `_extract_scenario_id` | `PatnaFlood2025`-style scenario identifiers |

## Real KG values used in this library

Worked examples reuse a small set of real entities pulled directly from the generated
`output/*.ttl` files, so every result shown is grounded in actual simulation output:

| Entity | Type | Flooded window | Peak depth | Notes |
| --- | --- | --- | --- | --- |
| `RoadSegment_000219a31d` | Road segment | `TimeInstant_011` → `TimeInstant_059` (continuous, 49 timesteps) | `30.0` m (Severe) | Velocity `5.0` m/s at `T011`/`T012` (above the `2.0` m/s risk threshold); subject of `FloodEvent_0000000` (`eventType "FloodStart"`, `eventID "EV_0000000"`) and `FloodSituation_00000` |
| `Building_00111dfad8` | Building | `TimeInstant_016` → `TimeInstant_054` (39 timesteps) | `31.0` m (Severe) | Severity passes through `Minor` → `Moderate` → `Severe` across its lifetime |
| `Hospital_0249c5e672` | Hospital (critical infrastructure) | `TimeInstant_013` → `TimeInstant_050` (38 timesteps) | `23.0` m (Moderate), reached at `TimeInstant_021` | Used for R-series/critical-infrastructure examples |

Severity bins (from `ONTOLOGY.md`): `Dry` (≤1.5 m) / `Minor` (≤15.0 m) / `Moderate` (≤29.0 m) /
`Severe` (>29.0 m). The simulation spans `TimeInstant_000`–`TimeInstant_059` (60 timesteps).

## Template index

| ID | Name | Series file |
| --- | --- | --- |
| S-01 | Was unit flooded at time T? | [S-series.md](S-series.md#s-01) |
| S-02 | Get depth at unit at time T | [S-series.md](S-series.md#s-02) |
| S-03 | Is unit safe now? | [S-series.md](S-series.md#s-03) |
| S-04 | List flooded units at time T | [S-series.md](S-series.md#s-04) |
| T-01 | When did flooding start? | [T-series.md](T-series.md#t-01) |
| T-02 | When did flooding end? | [T-series.md](T-series.md#t-02) |
| T-03 | How long was unit flooded? | [T-series.md](T-series.md#t-03) |
| T-04 | Full flood timeline | [T-series.md](T-series.md#t-04) |
| A-01 | Peak depth at unit | [A-series.md](A-series.md#a-01) |
| A-02 | Most severely affected units | [A-series.md](A-series.md#a-02) |
| A-03 | How many units flooded now? | [A-series.md](A-series.md#a-03) |
| E-01 | Did unit experience event? | [E-series.md](E-series.md#e-01) |
| E-03 | What events at unit? | [E-series.md](E-series.md#e-03) |
| R-01 | High velocity risk? | [R-series.md](R-series.md#r-01) |
| R-03 | Critical infrastructure at risk? | [R-series.md](R-series.md#r-03) |
| D-01 | Evacuation priorities | [D-series.md](D-series.md#d-01) |

## Classification pipeline

Every question runs through `_rule_based_classify` first, a deterministic, fast,
no-model-call set of substring/regex guardrails covering the most common phrasings. If no rule
matches, the question goes to Ollama via `build_llm_prompt`/`build_fast_llm_prompt`, which
embeds the same 16 `query_type` values plus a compact decision list and priority-disambiguation
examples (e.g. *"How long was building X flooded?" → T-03*). Together, the two stages give
every question a path to a template: common phrasings resolve instantly through the rule
cascade, and everything else is handled by the LLM using the same template vocabulary.
