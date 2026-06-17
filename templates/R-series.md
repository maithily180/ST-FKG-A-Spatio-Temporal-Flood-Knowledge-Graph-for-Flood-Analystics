# R-series: Risk Queries

> Flood-risk assessment based on hazardous flow velocities and critical infrastructure exposed
> to flood depths above defined thresholds.
> Defining example: *"Which critical infrastructure units are currently at flood risk?"*

There are **2 static templates** (`R-01`, `R-03`) and **2 dynamic patterns**.

---

## R-01

### High velocity risk?

**Purpose.** Check whether a unit experienced dangerous flow velocity. This is the "hazardous
flow velocities" half of this category's framing. (`human_intent`: *"Detect whether dangerous
flow velocity affected the selected unit."*)

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Retrieve linked `VelocityObservation` records.
3. Compare velocity values with the risk threshold.
4. Return whether dangerous-velocity evidence exists.

**Ontology elements touched:** `ex:SpatialUnit`, `ex:FloodUnitState`,
`ex:VelocityObservation` via `ex:ofUnit`, `ex:hasObservation`, `ex:velocityValue`.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `UNIT_ID` | **Required** | none |
| `VELOCITY_THRESHOLD` | Optional | `2.0` (m/s) |

**Rule-trigger phrase(s):** contains `"dangerous velocity"`, `"dangerous velocities"`,
`"high velocity"`, `"high velocities"`, `"velocity"`, or `"velocities"` → `R-01 / velocity_risk`
(`0.98`) for a single named unit, or the listing variant below for plural scope.

**SPARQL template**

```sparql
SELECT (COUNT(?obs) AS ?dangerObs)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:VelocityObservation ;
       ex:velocityValue ?vel .
  FILTER(xsd:float(?vel) > {VELOCITY_THRESHOLD})
}
```

**Worked example**

> NL question: *"Is RoadSegment_000219a31d at risk from high velocity?"*

```sparql
SELECT (COUNT(?obs) AS ?dangerObs)
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:VelocityObservation ;
       ex:velocityValue ?vel .
  FILTER(xsd:float(?vel) > 2.0)
}
```

Result: `dangerObs = 2`. This road segment's velocity readings of `5.0` m/s at both
`TimeInstant_011` and `TimeInstant_012` both clear the `2.0` m/s threshold.

#### Extended dynamic pattern: `velocity_risk_units`

**Trigger:** same velocity phrase list, with plural scope (e.g. a bare plural like "units" or
"roads", and no concrete unit named). Lists every unit exposed to dangerous velocity:

```sparql
SELECT DISTINCT ?unit
WHERE {
  ?state ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:VelocityObservation ;
       ex:velocityValue ?vel .
  FILTER(xsd:float(?vel) > 2.0)
}
LIMIT 10
```

> NL question: *"Which units face dangerous velocities?"*

Result includes `ex:RoadSegment_000219a31d` (velocity `5.0` m/s).

---

## R-03

### Critical infrastructure at risk?

**Purpose.** Find hospitals, schools, or power facilities currently exposed to flood depth
above the risk threshold. This is the "critical infrastructure exposed to flood depths above
defined thresholds" half of this category's framing, and the template this category's defining
example maps to directly.

**Logic / method:**
1. Restrict candidate units to hospitals, schools, and power facilities.
2. Use the current/default `TimeInstant`.
3. Retrieve each unit's `FloodUnitState` and depth observation.
4. Keep facilities whose depth exceeds the risk threshold.
5. Return exposed critical infrastructure.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `TIME_INSTANT` | Optional | `TimeInstant_011` |
| `DEPTH_THRESHOLD` | Optional | `0.1` |
| `LIMIT_VALUE` | Optional | `20` |

**Rule-trigger phrase(s):** contains `"critical infrastructure"`/`"critical infra"`/
`"hospital"`/`"school"`/`"power facility"`/`"power facilities"` **and** contains `"at risk"`/
`"risk"`/`"endangered"`/`"endanger"` → `R-03 / critical_infrastructure_risk` (`0.98`).

**SPARQL template**

```sparql
SELECT DISTINCT ?unit ?type
WHERE {
  BIND(ex:{TIME_INSTANT} AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?unit ex:type ?type .
  FILTER(?type IN ("Hospital", "PowerFacility", "School"))
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
}
LIMIT {LIMIT_VALUE}
```

**Worked example: this category's defining example, verbatim**

> NL question: *"Which critical infrastructure units are at risk at TimeInstant_013?"*

```sparql
SELECT DISTINCT ?unit ?type
WHERE {
  BIND(ex:TimeInstant_013 AS ?t)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?t ;
         ex:hasObservation ?obs .
  ?unit ex:type ?type .
  FILTER(?type IN ("Hospital", "PowerFacility", "School"))
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
LIMIT 20
```

Result row: `(ex:Hospital_0249c5e672, "Hospital")`. Depth `16.7347354888916` m at that
instant clears the `0.1` m threshold.

#### Extended dynamic pattern: narrower typed risk check (`R-03` + specific `ENTITY_TYPE`)

**Trigger:** the same rule above fires, but the question names a specific facility type (e.g.
"hospitals") rather than the umbrella phrase "critical infrastructure," so the query is scoped
to that one type instead of the `IN (...)` umbrella filter:

```sparql
SELECT DISTINCT ?unit
WHERE {
  BIND(ex:TimeInstant_013 AS ?t)
  ?unit ex:type "Hospital" .
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

> NL question: *"Which hospitals are at risk at TimeInstant_013?"*

Result: `ex:Hospital_0249c5e672`.
