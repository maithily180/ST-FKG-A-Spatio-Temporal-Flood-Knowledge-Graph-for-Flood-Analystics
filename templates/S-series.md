# S-series: State (Spatial) Queries

> Infrastructure flood-state retrieval and spatial flood-status analytics.
> Defining example: *"Which buildings were affected by the flood?"*

State queries answer "what is true about this unit (or these units) at a point in time?"
This is the most basic read against `ex:FloodUnitState`. Every other family builds on top of this one:
temporal queries aggregate state across time, aggregate queries rank state across units, event
queries explain *why* a state changed, risk/decision queries threshold state against
operational criteria. There are **4 static templates** (`S-01`–`S-04`) and **7 dynamic
patterns** built on top of `S-02`/`S-03`/`S-04`.

All four static templates and all dynamic patterns here operate on the same core graph shape:

```
?unit <-ex:ofUnit- ?state -ex:occursAt-> ?t
                     |
                     ex:hasObservation
                     v
                 ?obs -ex:depthValue-> ?depth
```

`FloodUnitState` nodes are materialized whenever a unit is flooded (depth > 0.1 m or velocity
> 0, see `ONTOLOGY.md` §3), which is what lets `S-01`/`S-03` answer presence/absence questions
with a simple `COUNT`.

---

## S-01

### Was unit flooded at time T?

**Purpose.** Determine whether a specific infrastructure unit was flooded at a selected time
instant. (`TEMPLATES["S-01"]["description"]`: *"Check if a specific unit was flooded at a given
time"*.)

**Logic / method** (from `explanation_builder.TEMPLATE_EXPLANATIONS["S-01"]`):
1. Bind the requested infrastructure unit.
2. Bind the requested `TimeInstant`.
3. Find the matching `FloodUnitState`.
4. Read the linked `DepthObservation`.
5. Compare the depth against the flood threshold.

The result is a `COUNT(?state)`: `1` if a matching, above-threshold state exists, `0`
otherwise.

**Ontology elements touched:** `ex:SpatialUnit`, `ex:FloodUnitState`, `ex:DepthObservation` via
`ex:ofUnit`, `ex:occursAt`, `ex:hasObservation`, `ex:depthValue`.

**Parameters**

| Parameter | Required? | Default | Notes |
| --- | --- | --- | --- |
| `UNIT_ID` | **Required** | none | e.g. `RoadSegment_000219a31d` |
| `TIME_INSTANT` | **Required** | none | e.g. `TimeInstant_011` |
| `DEPTH_THRESHOLD` | Optional | `0.1` | From `DEFAULT_PARAMS` if not stated/extracted |

**Rule-trigger phrase(s):** the question contains both `"was"` and `"flooded"`, **and** a time
instant was extracted → `S-01 / was_flooded` (confidence `0.97`).

**SPARQL template**

```sparql
SELECT (COUNT(?state) AS ?flooded)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  BIND(ex:{TIME_INSTANT} AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
}
```

**Worked example 1: flooded**

> NL question: *"Was RoadSegment_000219a31d flooded at TimeInstant_011?"*

Extracted params: `UNIT_ID=RoadSegment_000219a31d`, `TIME_INSTANT=TimeInstant_011`,
`DEPTH_THRESHOLD=0.1` (default).

```sparql
SELECT (COUNT(?state) AS ?flooded)
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?unit)
  BIND(ex:TimeInstant_011 AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
```

Result: `flooded = 1`. `TimeInstant_011` is this road segment's flood-arrival instant.

**Worked example 2: not yet flooded**

> NL question: *"Was Hospital_0249c5e672 flooded at TimeInstant_005?"*

`Hospital_0249c5e672`'s flood window begins at `TimeInstant_013`, so at `TimeInstant_005` the
query finds no matching state.

Result: `flooded = 0`.

---

## S-02

### Get depth at unit at time T

**Purpose.** Retrieve the exact depth value at a unit and time. (*"Retrieve the exact depth
value at a location and time"*.)

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Bind the requested `TimeInstant`.
3. Find the corresponding `FloodUnitState`.
4. Return the linked `DepthObservation` value.

**Ontology elements touched:** same as S-01.

**Parameters**

| Parameter | Required? | Default | Notes |
| --- | --- | --- | --- |
| `UNIT_ID` | **Required** | none | |
| `TIME_INSTANT` | Optional | `TimeInstant_011` | Falls back to the default reference instant if the question doesn't name one |

**Rule-trigger phrase(s):** `"depth"` in the question **and** a unit was extracted **and**
(a time instant was extracted, OR the question mentions "now"/"currently"/"current"/"right
now", OR it contains `"what is"`/`"what's"`) → `S-02 / get_depth` (confidence `0.92`).

**SPARQL template**

```sparql
SELECT ?depth
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  BIND(ex:{TIME_INSTANT} AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
```

**Worked example**

> NL question: *"What is the depth at Hospital_0249c5e672 at TimeInstant_013?"*

Params: `UNIT_ID=Hospital_0249c5e672`, `TIME_INSTANT=TimeInstant_013`.

```sparql
SELECT ?depth
WHERE {
  BIND(ex:Hospital_0249c5e672 AS ?unit)
  BIND(ex:TimeInstant_013 AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
```

Result: `depth = "16.7347354888916"^^xsd:float`, taken directly from
`ex:DepthObservation_0249c5e672_T013` in `output/observations.ttl`.

#### Extended dynamic pattern: `flood_trend`

**Trigger:** a unit is named **and** the question contains `"increasing or decreasing"` or
`"first vs last"`.

Returns the depth at a unit's *first* flooded instant and at its *last* flooded instant side by
side, showing the overall rise/fall shape of its flood episode:

```sparql
SELECT ?firstDepth ?lastDepth
WHERE {
  BIND(ex:{UNIT_ID} AS ?building)
  {
    SELECT (MIN(?time) AS ?tfirst)
    WHERE {
      ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?time ; ex:hasObservation ?o .
      ?o a ex:DepthObservation ; ex:depthValue ?d . FILTER(xsd:float(?d) > {DEPTH_THRESHOLD})
    }
  }
  {
    SELECT (MAX(?time) AS ?tlast)
    WHERE {
      ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?time ; ex:hasObservation ?o .
      ?o a ex:DepthObservation ; ex:depthValue ?d . FILTER(xsd:float(?d) > {DEPTH_THRESHOLD})
    }
  }
  ?s1 a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?tfirst ; ex:hasObservation ?o1 .
  ?o1 a ex:DepthObservation ; ex:depthValue ?firstDepth .
  ?s2 a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?tlast ; ex:hasObservation ?o2 .
  ?o2 a ex:DepthObservation ; ex:depthValue ?lastDepth .
}
```

> NL question: *"Is the depth increasing or decreasing for Building_00111dfad8?"*

`?tfirst` binds to `TimeInstant_016` and `?tlast` to `TimeInstant_054`. The building's
severity trajectory runs `Minor → Moderate → Severe` (peaking at `31.0` m) before receding, so
both `firstDepth` and `lastDepth` sit well below the peak, a rise-then-fall shape captured by
this two-point comparison.

---

## S-03

### Is unit safe now?

**Purpose.** Check if a unit is currently flooded, i.e. depth exceeds the threshold at the
reference time. (*"Check if a unit is currently flooded (depth > threshold)"*.)

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Use the current/default `TimeInstant`.
3. Retrieve depth evidence from the linked `FloodUnitState`.
4. Classify the unit as exposed if depth exceeds the threshold.

**Parameters**

| Parameter | Required? | Default | Notes |
| --- | --- | --- | --- |
| `UNIT_ID` | **Required** | none | |
| `TIME_INSTANT` | Optional | `TimeInstant_011` | |
| `DEPTH_THRESHOLD` | Optional | `0.1` | |

**Rule-trigger phrase(s):** the literal word `"safe"` anywhere in the question →
`S-03 / is_safe` (confidence `0.97`).

**Plural handling.** When the question asks about safety in plural scope (e.g. "are the
hospitals safe?") without naming one specific unit, the system answers using `S-04`'s listing
query instead, naturally extending "is X safe" into "which of these units are currently
flooded" for a group of units.

**SPARQL template**

```sparql
SELECT (COUNT(?state) AS ?floodCount)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  BIND(ex:{TIME_INSTANT} AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
}
```

**Worked example**

> NL question: *"Is RoadSegment_000219a31d safe right now?"*

Params: `UNIT_ID=RoadSegment_000219a31d`, `TIME_INSTANT=TimeInstant_011` (default reference
instant), `DEPTH_THRESHOLD=0.1`.

```sparql
SELECT (COUNT(?state) AS ?floodCount)
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?unit)
  BIND(ex:TimeInstant_011 AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
```

Result: `floodCount = 1` → **not safe** (flooded), consistent with `S-01`'s example above,
since `TimeInstant_011` is this road's flood-arrival instant.

#### Extended dynamic pattern: `continuous_check`

**Trigger:** a unit is named **and** the question contains `"flood continuously"` or
`"flooded continuously"`.

Checks whether every flooded timestep for the unit forms one unbroken run, by comparing the
count of distinct flooded instants against the count of instants that also have a successor
flooded instant:

```sparql
SELECT ?continuous
WHERE {
  BIND(ex:{UNIT_ID} AS ?building)
  {
    SELECT (IF(COUNT(?t1) = COUNT(?t2), true, false) AS ?continuous)
    WHERE {
      { SELECT ?t1 WHERE {
          ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?t1 ; ex:hasObservation ?o .
          ?o a ex:DepthObservation ; ex:depthValue ?d . FILTER(xsd:float(?d) > {DEPTH_THRESHOLD})
        }
        GROUP BY ?t1 }
      OPTIONAL {
        SELECT ?t2 WHERE {
          ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?t2 ; ex:hasObservation ?o2 .
          ?o2 a ex:DepthObservation ; ex:depthValue ?d2 . FILTER(xsd:float(?d2) > {DEPTH_THRESHOLD})
        }
        GROUP BY ?t2 }
    }
  }
}
```

> NL question: *"Did RoadSegment_000219a31d flood continuously?"*

This road's flooded instants run `TimeInstant_011` through `TimeInstant_059` with no gaps, so
this evaluates to `continuous = true`.

---

## S-04

### List flooded units at time T

**Purpose.** Get all units flooded at a specific time instant. (*"Get all units flooded at a
specific time instant"*.) This is the broadest static template in the S-series, with six
dynamic variants layered on top of it to cover typed, ranged, and scenario-scoped phrasings of
"which units…" questions.

**Logic / method:**
1. Use the current/default or requested `TimeInstant`.
2. Retrieve `FloodUnitState` records for candidate units.
3. Read each linked `DepthObservation`.
4. Return units whose depth exceeds the threshold.

**Parameters**

| Parameter | Required? | Default | Notes |
| --- | --- | --- | --- |
| `TIME_INSTANT` | Optional | `TimeInstant_011` | |
| `DEPTH_THRESHOLD` | Optional | `0.1` | |
| `LIMIT_VALUE` | Optional | `20` | |

Every parameter has a usable default, which is what makes `S-04` a natural target for plural
listing questions even when no other parameters are supplied.

**Rule-trigger phrase(s):**
- No unit named, the question mentions "now"/"currently"/"current"/"right now", is in plural
  scope, and contains "unsafe"/"flood"/"flooding"/"at risk"/"affected"/"impacted" →
  `S-04 / list_flooded` (`0.96`).
- Contains `"which"`/`"list"`/`"show"` **and** `"flooded"`/`"flooding"` →
  `S-04 / list_flooded` (`0.92`).

**SPARQL template**

```sparql
SELECT DISTINCT ?unit
WHERE {
  BIND(ex:{TIME_INSTANT} AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
}
LIMIT {LIMIT_VALUE}
```

**Worked example**

> NL question: *"Which units are flooded right now?"*

Params: no `TIME_INSTANT` extracted → defaults to `TimeInstant_011`; `DEPTH_THRESHOLD=0.1`;
`LIMIT_VALUE=20`.

```sparql
SELECT DISTINCT ?unit
WHERE {
  BIND(ex:TimeInstant_011 AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
LIMIT 20
```

Result: up to 20 distinct units, including `ex:RoadSegment_000219a31d`.

#### Extended dynamic patterns

**1. Typed listing (`S-04` + `ENTITY_TYPE`).** When a category like "buildings" is named, an
inline type filter is added:

> NL question: *"Which buildings are flooded right now?"* → `ENTITY_TYPE=Building` extracted.

```sparql
SELECT DISTINCT ?unit
WHERE {
  BIND(ex:TimeInstant_011 AS ?t)
  ?unit ex:type "Building" .
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
LIMIT 20
```

(For `ENTITY_TYPE=CriticalInfrastructure`, the filter becomes
`FILTER(?entityType IN ("Hospital", "PowerFacility", "School"))`.)

**2. `affected_entities_overall`: this category's defining example.** Trigger: the question
contains `"became impacted"` or `"were affected"` **and** an `ENTITY_TYPE` was extracted.

> NL question: *"Which buildings were affected by the flood?"* → `ENTITY_TYPE=Building`.

This asks "was this unit *ever* flooded, at any point in the simulation?" rather than at one
instant:

```sparql
SELECT DISTINCT ?unit
WHERE {
  ?unit a ex:SpatialUnit ;
        ex:type "Building" .
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit .
}
```

**3. `scenario_membership`.** Trigger: a `PatnaFlood\d{4}`-style `SCENARIO_ID` is named **and**
the question contains "part of"/"belong to"/"in patnaflood".

> NL question: *"Which units are part of PatnaFlood2025?"*

```sparql
SELECT ?unit
WHERE {
  ex:PatnaFlood2025 ex:affects ?unit .
}
```

`ex:FloodSituation_00000` plays this `ex:affects` role for `ex:RoadSegment_000219a31d`
specifically; the global `PatnaFlood2025` scenario aggregates across all per-unit situations.

**4. `snapshot_at_time`.** Trigger: a time instant is named **and** the question contains
"what is happening"/"what's happening"/"happening at".

> NL question: *"What's happening at TimeInstant_013?"*

Returns `(?unit, ?depth)` pairs, a full situation report across all flooded units at that
instant:

```sparql
SELECT ?unit ?depth
WHERE {
  BIND(ex:TimeInstant_013 AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
LIMIT 20
```

Real-KG row: `(ex:Hospital_0249c5e672, "16.7347354888916"^^xsd:float)`.

**5. `range_became_flooded`.** Trigger: two time instants are named **and** the question
contains "between"/"became flooded".

> NL question: *"Which units became flooded between TimeInstant_011 and TimeInstant_013?"*

```sparql
SELECT DISTINCT ?unit
WHERE {
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  FILTER(?time >= ex:TimeInstant_011 && ?time <= ex:TimeInstant_013)
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
LIMIT 50
```

`ex:RoadSegment_000219a31d` (flooded since `T011`) and `ex:Hospital_0249c5e672` (flooded since
`T013`) are both members of this result, since both have a qualifying state in the
`[T011, T013]` window.

**6. `flood_situation`.** Trigger: a unit is named **and** the question contains
"flood situation". Pivots from state-level data to the `FloodSituation` provenance node:

> NL question: *"What is the flood situation for RoadSegment_000219a31d?"*

```sparql
SELECT ?situation
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?building)
  ?situation a ex:FloodSituation ;
             ex:affects ?building .
}
```

Result: `?situation = ex:FloodSituation_00000`.
