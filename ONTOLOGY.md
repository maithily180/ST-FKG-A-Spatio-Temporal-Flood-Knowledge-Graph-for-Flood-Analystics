# ST-FKG Ontology

This document explains the ST-FKG (Spatio-Temporal Flood Knowledge Graph) ontology in
detail: what it models, how its classes and properties fit together, and the standards
alignment added on top of the original lightweight schema. The machine-readable file is
[`ontology/stfkg_ontology.rdf`](ontology/stfkg_ontology.rdf) (OWL/XML).

## Diagram

![ST-FKG Ontology diagram](./Median%20Value%20Decision%20Tree-2026-06-17-123036.png)

## 1. Design philosophy

ST-FKG models **infrastructure-centric flood state**, not raw rasters. A flood simulation
produces a depth/velocity raster per timestep; the ontology's job is to turn "this pixel is
1.2 m deep" into "this building was Moderately flooded at this time, with this depth and
this velocity, as part of this event and this overall situation" — i.e. a graph of typed,
queryable entities rather than an opaque grid.

The ontology is intentionally **small and project-specific** rather than a heavyweight reuse
of every relevant W3C/OGC standard. Three external vocabularies are reused where they map
cleanly onto what is actually produced by the pipeline:

| Reused vocabulary | Used for | Status before this revision | Status after this revision |
| --- | --- | --- | --- |
| **GeoSPARQL** (`geo:`) | Feature/Geometry, WKT literals | Implemented | Implemented (unchanged) |
| **QUDT** (`qudt:`, `unit:`, `quantitykind:`) | Quantity/unit modeling for depth, velocity, resolution | Not implemented (namespace not even declared) | Implemented, additive |
| **W3C Time Ontology** (`time:`) | Temporal entity typing for instants/intervals | Namespace declared but unused/dangling | Implemented, additive |

Everything else — `SpatialUnit`, `FloodUnitState`, `FloodEvent`, `FloodSituation`,
`Observation`, `GeoCoverage` and their properties — is a **custom domain vocabulary** under
the base namespace:

```
http://www.semanticweb.org/maith/ontologies/2026/2/untitled-ontology-2#
```

conventionally bound to the prefix `ex:`. There is no SOSA/SSN reuse in the current
implementation; that remains a documented future-work item, not something claimed as done.

## 2. Namespaces

| Prefix | IRI | Role |
| --- | --- | --- |
| `ex:` | `http://www.semanticweb.org/maith/ontologies/2026/2/untitled-ontology-2#` | Project domain vocabulary |
| `geo:` | `http://www.opengis.net/ont/geosparql#` | Spatial features and geometry |
| `time:` | `http://www.w3.org/2006/time#` | Temporal entity typing (newly wired in) |
| `qudt:` | `http://qudt.org/schema/qudt/` | Quantity/Unit/QuantityKind modeling (newly wired in) |
| `unit:` | `http://qudt.org/vocab/unit/` | QUDT unit individuals (`unit:M`, `unit:M-PER-SEC`) |
| `quantitykind:` | `http://qudt.org/vocab/quantitykind/` | QUDT quantity-kind individuals (`Length`, `Velocity`) |
| `xsd:` | `http://www.w3.org/2001/XMLSchema#` | Literal datatypes |
| `owl:` / `rdfs:` / `rdf:` | standard | OWL/RDFS/RDF meta-vocabulary |

## 3. Class model

```
geo:Feature
  └─ ex:SpatialUnit            (Building, RoadSegment, RailwaySegment, Hospital, School, PowerFacility)

ex:FloodUnitState              (one node per spatial unit per flooded timestep)

ex:Observation
  ├─ ex:DepthObservation
  └─ ex:VelocityObservation

ex:FloodEvent                  (FloodStart, FloodEnd, PeakFlood, PersistentFlood,
                                 SeverityChange, RapidRise, RapidFall, HighVelocity,
                                 CriticalInfrastructureImpact — typed via ex:eventType literal,
                                 not via OWL subclasses)

ex:FloodSituation              (per-unit and one global ex:PatnaFlood2025 situation)

ex:GeoCoverage                 (one node per source raster file)

geo:Geometry                   (WKT geometry of a SpatialUnit)

time:TemporalEntity
  ├─ time:Instant  ← ex:TimeInstant
  └─ time:Interval
       └─ time:ProperInterval  ← ex:TimeInterval

qudt:QuantityValue             (new: pairs a numeric value with a unit and quantity kind)
qudt:Unit                      (new: unit:M, unit:M-PER-SEC)
qudt:QuantityKind              (new: quantitykind:Length, quantitykind:Velocity)
```

`ex:SpatialUnit` is the only class that subclasses an external type directly
(`geo:Feature`), which is what makes `geo:hasGeometry` / `geo:asWKT` queries against
buildings, roads, hospitals, etc. work. `FloodUnitState`, `FloodEvent`, `FloodSituation`,
`Observation` and `GeoCoverage` are root classes in the domain vocabulary — there was no
existing standard class that cleanly matched "a unit's flood state at a timestep," so the
project defines its own rather than forcing a strained reuse.

### Why `FloodUnitState` is the central node

Flood attributes are never attached directly to a `SpatialUnit`. Instead:

```
SpatialUnit ←ex:ofUnit— FloodUnitState —ex:occursAt→ TimeInstant
                              │
                              ex:hasObservation
                              ↓
                  DepthObservation, VelocityObservation
```

This indirection is what makes the model spatio-*temporal*: a building doesn't have "a
depth," it has many `FloodUnitState` nodes, one per timestep it was flooded, each carrying
its own observations, severity and links to events. The graph is **sparse by construction**
— states are only materialized where `depth > threshold OR velocity > 0`, so "is this unit
ever safe" is answered by absence of a state, not by a `False` flag.

## 4. Object properties

| Property | Domain | Range | Meaning |
| --- | --- | --- | --- |
| `ex:ofUnit` | FloodUnitState | SpatialUnit | which unit this state belongs to |
| `ex:hasObservation` | FloodUnitState | Observation | the depth/velocity readings for this state |
| `ex:occursAt` | FloodUnitState ∪ FloodEvent | TimeInstant | when this state/event happened |
| `ex:continuesAs` | FloodUnitState | FloodUnitState | links a unit's state to its next state in time (built by `05_link_state_transitions.py`) |
| `ex:derivedFromCoverage` | Observation | GeoCoverage | provenance: which raster file this observation came from |
| `ex:derivedFromState` | FloodEvent | FloodUnitState | provenance: which state(s) triggered this event |
| `ex:impacts` | FloodEvent | SpatialUnit | which unit an event impacted |
| `ex:affects` | FloodSituation | SpatialUnit | which units a situation covers |
| `ex:involves` | FloodSituation | FloodEvent | which events belong to a situation |
| `ex:hasTime` | FloodSituation | TimeInterval | the situation's overall time span |
| `ex:hasBeginning` / `ex:hasEnd` | TimeInterval | TimeInstant | interval bounds (now `rdfs:subPropertyOf time:hasBeginning` / `time:hasEnd`) |
| `geo:hasGeometry` | SpatialUnit | geo:Geometry | spatial footprint |
| `ex:hasQuantityValue` *(new)* | — *(intentionally unconstrained)* | qudt:QuantityValue | bridge from a measurement-bearing node to its QUDT representation |

## 5. Datatype properties

| Property | Domain | Range | Notes |
| --- | --- | --- | --- |
| `ex:depthValue` | DepthObservation | `xsd:float` | meters; read directly by every SPARQL template that filters/ranks by depth |
| `ex:velocityValue` | VelocityObservation | `xsd:float` | m/s |
| `ex:severityLevel` | FloodUnitState | `xsd:string` | `Dry` / `Minor` / `Moderate` / `Severe`, thresholded at 1.5 / 15.0 / 29.0 m |
| `ex:stateID`, `ex:eventID`, `ex:situationID`, `ex:unitID` | respective classes | `xsd:string` | human-readable identifiers, separate from the RDF local name |
| `ex:eventType` | FloodEvent | `xsd:string` | e.g. `RapidRise`, `PersistentFlood`, `CriticalInfrastructureImpact` |
| `ex:type` | SpatialUnit | `xsd:string` | `Building` / `RoadSegment` / `Hospital` / `School` / `PowerFacility` / `RailwaySegment` |
| `ex:fileName` | GeoCoverage | `xsd:string` | source raster file, e.g. `depth_20250801_0600.tif` |
| `ex:resolution` | GeoCoverage | `xsd:string` | e.g. `"5m"` — magnitude and unit concatenated in one literal (see §6) |
| `ex:timestamp` | TimeInstant | `xsd:dateTime` | now `rdfs:subPropertyOf time:inXSDDateTime` |
| `geo:asWKT` | geo:Geometry | `geo:wktLiteral` | the unit's footprint as WKT |

## 6. What changed: QUDT alignment

**Problem.** Depth and velocity were stored as bare floats with the unit only documented in
prose (the README), never in the graph itself: `ex:depthValue "19.89"^^xsd:float` says
nothing machine-readable about whether that's meters, feet, or centimeters. `ex:resolution`
was worse — the unit was baked into the string itself (`"5m"`), which is not a queryable
number at all.

**Fix.** Every measurement now has a complementary, fully QUDT-compliant representation
reachable via the new `ex:hasQuantityValue` property, without touching the existing literal
properties that the deployed SPARQL templates already depend on. Concretely, for the real
KG fragment

```turtle
ex:DepthObservation_000219a31d_T011 a ex:DepthObservation ;
    ex:depthValue "19.890213012695312"^^xsd:float ;
    ex:derivedFromCoverage ex:DepthCoverage_011 .
```

the QUDT-aligned addition (instance data, not shown in the ontology file itself — the
ontology only declares the *schema* for this pattern) would look like:

```turtle
ex:DepthObservation_000219a31d_T011
    ex:hasQuantityValue [
        a qudt:QuantityValue ;
        qudt:numericValue "19.890213012695312"^^xsd:double ;
        qudt:unit unit:M ;
        qudt:hasQuantityKind quantitykind:Length
    ] .
```

and for velocity, the same shape with `qudt:unit unit:M-PER-SEC` and
`quantitykind:Velocity`. `ex:resolution "5m"` becomes (conceptually — the property's
declared range is left as `xsd:string` to avoid breaking already-exported `coverage.ttl`
literals) a `qudt:QuantityValue` with `qudt:numericValue "5"^^xsd:double` and
`qudt:unit unit:M`.

This is the standard QUDT pattern — pair a number (`qudt:numericValue`) with a unit
(`qudt:unit`) and a quantity kind (`qudt:hasQuantityKind`) — reused exactly as published at
`http://qudt.org/2.1/schema/qudt`, `.../vocab/unit`, `.../vocab/quantitykind`. The ontology
file declares `owl:imports` for those three IRIs **and** locally stub-declares the classes,
properties and unit/quantity-kind individuals it actually uses
(`qudt:QuantityValue`, `qudt:Unit`, `qudt:QuantityKind`, `qudt:numericValue`, `qudt:unit`,
`qudt:hasQuantityKind`, `unit:M`, `unit:M-PER-SEC`, `quantitykind:Length`,
`quantitykind:Velocity`), so the file remains valid and loadable even with no network access
to resolve the imports.

**Why additive, not a replacement.** The deployed query executor
(`stfkg_query_executor.py`) generates SPARQL like
`FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})` directly against `ex:depthValue`. Retyping or
removing that property would break every existing template. The QUDT structure is therefore
a second, standards-compliant view of the same measurement, not a migration — generating it
for the live KG is a follow-up data-export step (extending `04_build_flood_states.py` /
`02_build_temporal_layer.py`), not an ontology change.

## 7. What changed: W3C Time Ontology alignment

**Problem.** The ontology file declared `xmlns:time="http://www.w3.org/2006/time#"` but
never used it anywhere — `ex:TimeInstant` and `ex:TimeInterval` were plain local classes with
no relationship to the standard temporal vocabulary the project's own documentation and the
accompanying paper claim to use.

**Fix.**

- `ex:TimeInstant rdfs:subClassOf time:Instant`
- `ex:TimeInterval rdfs:subClassOf time:ProperInterval` (a `time:Interval` with distinct,
  non-equal beginning/end — true for every `ex:TimeInterval` in the KG, which always chains
  two distinct instants)
- `ex:hasBeginning rdfs:subPropertyOf time:hasBeginning`
- `ex:hasEnd rdfs:subPropertyOf time:hasEnd`
- `ex:timestamp rdfs:subPropertyOf time:inXSDDateTime`

Because these are `subClassOf`/`subPropertyOf` axioms rather than replacements, every
existing triple keeps working unmodified; an RDFS-aware reasoner or triple store additionally
entails the standard-vocabulary triple (e.g. a `ex:timestamp` triple also counts as a
`time:inXSDDateTime` triple). `owl:imports` for `http://www.w3.org/2006/time` is declared,
with local stub declarations of `time:TemporalEntity`, `time:Instant`, `time:Interval`,
`time:ProperInterval`, `time:hasBeginning`, `time:hasEnd`, `time:inXSDDateTime` for the same
offline-safety reason as the QUDT stubs.

## 8. Backward compatibility summary

Nothing that existed before this revision was removed, renamed, or retyped:

- All original classes, object properties and datatype properties keep their original
  `rdfs:domain`/`rdfs:range`.
- `ex:depthValue`, `ex:velocityValue`, `ex:resolution` keep their original ranges
  (`xsd:float`, `xsd:float`, `xsd:string`).
- No existing `.ttl` instance data needs to change for the ontology file to remain valid
  against it.
- Every addition is either a brand-new term (`ex:hasQuantityValue`, the QUDT/Time stub
  classes and properties, the unit/quantity-kind individuals) or a `subClassOf`/
  `subPropertyOf` axiom layered on an existing term.

## 9. Known limitations (stated plainly)

- The QUDT `qudt:QuantityValue` instance pattern is defined at the **schema** level in
  `stfkg_ontology.rdf`; the **instance-level** KG export scripts (`scripts/04_*`,
  `scripts/02_*`) do not yet emit those triples. Until that export step is added, a triple
  store loaded only with the existing pipeline output will have the QUDT *classes and
  properties* available but no `qudt:QuantityValue` *individuals* in the data.
- `ex:resolution`'s declared range is still `xsd:string` (e.g. `"5m"`) for backward
  compatibility with already-exported `coverage.ttl` files; only the *complementary* QUDT
  view treats it as a proper Length quantity.
- SOSA/SSN observation modeling is still not implemented or claimed as implemented anywhere
  in this revision — only QUDT and W3C Time were in scope.
- `owl:imports` targets (`qudt.org`, `w3.org/2006/time`) require network access to fully
  resolve in a reasoner; the file is deliberately self-contained (via local stub
  declarations) so it loads and is queryable without that resolution.

## 10. Loading and validating the file

```python
import rdflib
g = rdflib.Graph()
g.parse("ontology/stfkg_ontology.rdf", format="xml")
print(len(g), "triples")
```

In Protège, disable "auto-load imported ontologies" (or work offline) if you don't want it
to attempt to fetch the QUDT/Time IRIs over the network — the file is fully valid and
browsable either way because of the local stub declarations described in §6–7.
