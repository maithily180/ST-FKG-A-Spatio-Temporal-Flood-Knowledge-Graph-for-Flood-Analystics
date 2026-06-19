# ST-FKG: A Spatio-Temporal Flood Knowledge Graph for Flood Analytics

ST-FKG turns raw, raster-based flood-simulation output into a structured, queryable
**knowledge graph of infrastructure flood state** — and then into natural-language answers,
with an explanation attached to every answer.

A hydrodynamic flood simulation produces a depth/velocity raster for every timestep: a grid
of numbers with no notion of "building," "road," or "hospital." On its own, that raster
cannot answer an operational question like *"which buildings remained flooded throughout the
event?"* or *"when did flooding first reach this hospital?"* — answering those requires
knowing which pixels overlap which real-world structures, and tracking how that overlap
evolves over time. ST-FKG closes that gap. It intersects multi-temporal flood rasters with
real infrastructure geometries (buildings, roads, railways, hospitals, schools, power
facilities), and represents the result as a graph: every infrastructure unit gets a sequence
of typed, timestamped `FloodUnitState` nodes carrying depth, velocity, severity, and event
information, rather than a single static "flooded: yes/no" flag. That graph is then queryable
in two ways — directly via SPARQL/GeoSPARQL, and through a hybrid rule-first/LLM-assisted
natural-language layer that maps plain-English questions onto one of 16 deterministic query
templates, so every answer is reproducible and every answer's reasoning can be inspected
rather than trusted blindly.

This repository is the public ontology + query-template artifact backing that system: it
documents *what the knowledge graph contains* (the ontology) and *how it is queried* (the
template library), independent of the simulation/ingestion pipeline and the Streamlit
application that consume them. It accompanies the paper *"ST-FKG: A Spatio-Temporal Flood
Knowledge Graph for Explainable Natural-Language-based Flood Analytics."*

## Why a knowledge graph, and why this design

Three design decisions shape everything in this repo:

- **Flood state lives on its own node, not on the building.** A `SpatialUnit` (a building,
  road, hospital, ...) never carries a depth or velocity attribute directly. Instead, every
  timestep at which a unit is flooded gets its own `FloodUnitState` node, linked to the unit,
  to a `TimeInstant`, and to its supporting `DepthObservation`/`VelocityObservation`. This
  indirection is what makes the graph *spatio-temporal* — a unit's flood history is a chain of
  states, not a single overwritten value — and it makes the graph **sparse by construction**:
  a unit that was never flooded simply has no `FloodUnitState` nodes, rather than a `false`
  flag sitting in storage.
- **Three established standards are reused where they genuinely fit, instead of building
  everything from scratch.** GeoSPARQL gives every `SpatialUnit` a real, queryable geometry.
  The W3C Time Ontology gives `TimeInstant`/`TimeInterval` a standard temporal type. QUDT
  gives every depth/velocity/resolution value a machine-readable unit and quantity kind,
  instead of leaving the unit implicit in a code comment. Everything else (`FloodUnitState`,
  `FloodEvent`, `FloodSituation`, `Observation`, `GeoCoverage`) is a small, project-specific
  vocabulary, because no existing standard class cleanly matches "a unit's flood condition at
  one timestep."
- **Natural-language access is deterministic, not generative.** Free-form LLM-to-SPARQL
  generation is fast to prototype but prone to hallucinated predicates and malformed queries
  against a domain-specific schema. This project instead classifies a question into one of 16
  fixed templates (first by deterministic phrase rules, falling back to an LLM only to
  *classify intent*, never to write SPARQL) and fills in the matching, pre-validated query.
  The query templates and the deterministic-classification rules that select them are exactly
  what's documented in `templates.md` and the six `templates/*.md` files.

## Repository contents

```
.
├── README.md             ← you are here
├── ONTOLOGY.md           ← full ontology documentation: classes, properties, design rationale
├── ontology/
│   ├── stfkg_ontology.rdf   ← the machine-readable ontology itself (OWL/XML)
│   └── ontology.png          ← class/property diagram referenced from ONTOLOGY.md
├── templates.md           ← index and shared reference for the SPARQL query template library
├── templates/
│   ├── S-series.md        ← State queries      — flood-state retrieval for a unit/time
│   ├── T-series.md        ← Temporal queries    — arrival, recession, duration, full timeline
│   ├── A-series.md        ← Aggregate queries   — ranking and counting across many units
│   ├── E-series.md        ← Event queries       — named flood-event occurrence and history
│   ├── R-series.md        ← Risk queries        — hazardous velocity / critical-infrastructure exposure
│   └── D-series.md        ← Decision queries    — evacuation and intervention prioritization
└── walkthroughs/
    ├── README.md                  ← the classification pipeline + how confidence is assigned
    ├── example-rules-based.md     ← canonical query traced through the deterministic rule path
    └── example-llm-paraphrase.md  ← paraphrased query traced through the LLM fallback path
```

### `ONTOLOGY.md` and `ontology/`

[`ONTOLOGY.md`](ONTOLOGY.md) is the authoritative description of the knowledge graph's
schema: every class, every object/datatype property, the namespaces in use, the reasoning
behind the `FloodUnitState`-centered design, exactly how GeoSPARQL/QUDT/W3C-Time are wired in
(and, just as importantly, what is *not* yet implemented — stated plainly in its "Known
limitations" section rather than glossed over). [`ontology/stfkg_ontology.rdf`](ontology/stfkg_ontology.rdf)
is the actual ontology file — load it with `rdflib`, Protégé, or any OWL-aware tool to browse
or extend the schema directly. [`ontology/ontology.png`](ontology/ontology.png) is the visual
class diagram embedded at the top of `ONTOLOGY.md`.

### `templates.md` and `templates/`

[`templates.md`](templates.md) is the index for the query-template library: it explains the
two layers of query pattern (16 static templates vs. their dynamic intent variants), how
parameters are extracted from a natural-language question, the default parameter values used
when a question doesn't specify one, and the real knowledge-graph entities reused across every
worked example so that results in the docs are grounded in actual simulation output rather than
invented numbers. Each of the six `templates/*-series.md` files documents one query family in
full: every template's purpose, its required/optional parameters, the literal phrases that
trigger deterministic (no-LLM) classification, the raw SPARQL with its `{PARAM}` placeholders,
and a fully worked example with real entity IDs.

### `walkthroughs/`

[`walkthroughs/`](walkthroughs/) traces, step by step, how a natural-language question is
classified into a template and how a confidence score is attached to that decision. Its
[`README.md`](walkthroughs/README.md) explains the two-stage pipeline (deterministic rules
first, LLM fallback second), the 0.75 confidence threshold, and the two distinct ways
confidence is assigned (hardcoded per-rule constants vs. the model's self-report). The two
example files trace the same duration question through both paths:
[`example-rules-based.md`](walkthroughs/example-rules-based.md) for the canonical phrasing
*"How long was Building_001 flooded?"* (matched by a deterministic rule, no LLM call), and
[`example-llm-paraphrase.md`](walkthroughs/example-llm-paraphrase.md) for the paraphrase
*"What period of inundation was sustained by Building_001?"* (no rule matches, so it falls
through to the LLM). Both converge on the same template and answer, illustrating why
classification only ever selects a template while the SPARQL itself is always generated
deterministically.

## How the pieces fit together

1. A natural-language question arrives (e.g. *"How long was Building_001 flooded?"*).
2. It is classified — deterministically if a known trigger phrase matches, otherwise by an
   LLM restricted to choosing among the same 16 template IDs documented in `templates/` — into
   exactly one template (here, `T-03`).
3. Parameters are extracted from the question text (unit ID, time instant, thresholds, ...)
   using the extractors documented in `templates.md`.
4. The matching SPARQL template from the relevant `templates/*-series.md` file is instantiated
   with those parameters and executed against the knowledge graph described in `ONTOLOGY.md`.
5. The result, together with the template used, the reasoning steps, and the supporting
   evidence, is returned — so the same question always produces the same answer, traceable
   back to a specific template and a specific set of graph triples.

## Status and scope

This repository documents the **ontology and query-template layer** of ST-FKG. It does not
include the raster-ingestion/simulation pipeline or the natural-language application
front-end; it is the schema and query contract those components are built against. Known gaps
and partially implemented pieces (e.g. QUDT instance-level export, SOSA/SSN reuse) are called
out explicitly in `ONTOLOGY.md` §9 rather than left implicit.
