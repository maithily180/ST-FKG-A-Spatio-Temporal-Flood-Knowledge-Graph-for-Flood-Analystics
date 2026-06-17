# A-series: Aggregate Queries

> Infrastructure ranking and flood-severity aggregation.
> Defining example: *"List the roads in the area in descending order of flood duration."*

Aggregate queries summarize flood severity **across many units at once**, ranking by peak
depth, by flood duration, by event diversity, or by simple counts, rather than reporting on
a single unit in isolation. The ranking workhorse of this family is `A-02`, which every
"which units were worst affected / which units flooded longest / rank the roads by depth"
style question ultimately resolves to. There are **3 static templates** (`A-01`-`A-03`) and
**8 dynamic patterns**, most of them extending `A-02` with a specific ranking metric or scope.

---

## A-01

### Peak depth at unit

**Purpose.** Get the maximum flooding depth a unit ever experienced, across the whole
simulation. (`human_intent`: *"Find the highest flood depth observed for a selected unit."*)

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Retrieve all depth observations linked through `FloodUnitState` records.
3. Convert depth values to numeric form.
4. Return the maximum observed depth.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `UNIT_ID` | **Required** | none |

**Rule-trigger phrase(s):**
- A unit is named, and the question contains `"peak"`/`"maximum"`/`"max"` **and**
  `"depth"`/`"water depth"` → `A-01 / peak_depth` (`0.97`).
- A unit is named, a `DEPTH_THRESHOLD` was extracted from the text, **and** the question
  contains `"severe flooding"`/`"severe"`/`"experience"`/`"experienced"`/`"exceed"`/
  `"above"`/`"over"` → `A-01 / peak_depth` (`0.95`).
- A unit is named **and** the question contains `"high risk"` together with either an
  extracted `DEPTH_THRESHOLD`, or `">"`/`"duration"`/`"peak"` → `A-01 / high_risk_check`
  (`0.98`), the richer combined depth-and-duration check described below.

**SPARQL template**

```sparql
SELECT (MAX(xsd:float(?depth)) AS ?peakDepth)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
```

**Worked example**

> NL question: *"What is the peak depth at RoadSegment_000219a31d?"*

```sparql
SELECT (MAX(xsd:float(?depth)) AS ?peakDepth)
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
```

Result: `peakDepth = 30.0`, this road segment's maximum recorded flood depth, classified as
`Severe` under the project's severity bins.

#### Extended dynamic pattern: `high_risk_check`

This variant checks **both** peak depth and flood duration against thresholds in one query,
identifying units that are both deep *and* long-lasting. If the caller doesn't supply one,
`DEPTH_THRESHOLD` defaults to `2.0` m for this pattern (instead of the usual `0.1`), and
`MIN_DURATION` defaults to `20` timesteps.

```sparql
SELECT ?building ?peak ?duration
WHERE {
  BIND(ex:{UNIT_ID} AS ?building)
  { SELECT ?building (MAX(xsd:float(?depth)) AS ?peak)
    WHERE {
      ?state a ex:FloodUnitState ; ex:ofUnit ?building ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
    }
    GROUP BY ?building
  }
  { SELECT ?building (COUNT(?state) AS ?duration)
    WHERE {
      ?state a ex:FloodUnitState ; ex:ofUnit ?building ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
      FILTER(xsd:float(?depth) > 0.1)
    }
    GROUP BY ?building
  }
  FILTER(?peak > {DEPTH_THRESHOLD} && ?duration > {MIN_DURATION})
}
```

> NL question: *"Is RoadSegment_000219a31d at high risk (depth > 2.0)?"*

Real-KG evaluation: `peak = 30.0` (above the `2.0` m threshold) and `duration = 49` (its count
of flooded states from `TimeInstant_011` through `TimeInstant_059`, above the `20`-timestep
minimum). Both conditions hold, so the road segment is returned as a high-risk result.

---

## A-02

### Most severely affected units

**Purpose.** Rank every unit, optionally restricted to a type, by its own maximum observed
depth, worst-first. This is the family's ranking workhorse, answering questions like
*"List the roads in descending order of flood duration"* or *"Which units were worst
affected?"*

**Logic / method:**
1. Retrieve candidate infrastructure units.
2. Find their associated `FloodUnitState` observations.
3. Extract each unit's maximum observed flood depth.
4. Sort units in descending severity order.
5. Return the top-ranked results.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `LIMIT_VALUE` | Optional | `20` |

**Rule-trigger phrase(s):** contains `"worst affected"`, `"most severely affected"`,
`"severely affected"`, `"most affected"`, `"worst"`, `"max depth per unit"`, or
`"peak flooding"` → `A-02 / worst_affected` (`0.96`).

**SPARQL template**

```sparql
SELECT ?unit (MAX(xsd:float(?depth)) AS ?maxDepth)
WHERE {
  ?state ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
GROUP BY ?unit
ORDER BY DESC(?maxDepth)
LIMIT {LIMIT_VALUE}
```

**Worked example**

> NL question: *"Which units are the worst affected?"*

```sparql
SELECT ?unit (MAX(xsd:float(?depth)) AS ?maxDepth)
WHERE {
  ?state ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
GROUP BY ?unit
ORDER BY DESC(?maxDepth)
LIMIT 20
```

Top of the real-KG-grounded result: `ex:Building_00111dfad8 → 31.0`, then
`ex:RoadSegment_000219a31d → 30.0`.

**Worked example: restricted to a type**

> NL question: *"Which roads were the worst affected?"* → `ENTITY_TYPE=RoadSegment` is
> extracted from "roads", restricting the ranking to road segments only.

#### Extended dynamic patterns

**0. Typed ranking (`A-02` + `ENTITY_TYPE`).** When a category like "roads" or "buildings" is
named, the ranking is automatically scoped to that type:

```sparql
SELECT ?unit (MAX(xsd:float(?depth)) AS ?maxDepth)
WHERE {
  ?unit ex:type "RoadSegment" .
  ?state ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
GROUP BY ?unit
ORDER BY DESC(?maxDepth)
LIMIT 20
```

Real-KG result: `ex:RoadSegment_000219a31d → 30.0`, ranked among road segments only.

**1. `continuous_flood_units`.** Trigger: plural scope **and** the question contains
`"remained flooded continuously"` or `"flooded continuously"`. Ranks units by how many flooded
timesteps they accumulated, keeping only those above 10:

```sparql
SELECT ?unit (COUNT(?state) AS ?floodCount)
WHERE {
  ?state ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
GROUP BY ?unit
HAVING(?floodCount > 10)
ORDER BY DESC(?floodCount)
LIMIT 10
```

> NL question: *"Which units remained flooded continuously?"*

Real-KG result: `RoadSegment_000219a31d → 49`, `Building_00111dfad8 → 39`,
`Hospital_0249c5e672 → 38`. All three qualify and are ranked road, building, hospital.

**2. `longest_building_durations`.** Trigger: contains `"longer flooding"` **and**
(`"which building"` or `"which buildings"`). Ranks buildings by flood duration in timesteps
(end minus start):

```sparql
SELECT ?building ?duration
WHERE {
  {
    SELECT ?building (xsd:integer(REPLACE(STR(MAX(?t)), ".*_(\\d+)$", "$1")) - xsd:integer(REPLACE(STR(MIN(?t)), ".*_(\\d+)$", "$1")) AS ?duration)
    WHERE {
      ?building ex:type "Building" .
      ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?t ; ex:hasObservation ?o .
      ?o a ex:DepthObservation ; ex:depthValue ?d .
      FILTER(xsd:float(?d) > 0.1)
    }
    GROUP BY ?building
  }
}
ORDER BY DESC(?duration)
LIMIT 10
```

> NL question: *"Which buildings had longer flooding durations?"*

Real-KG row: `ex:Building_00111dfad8 → 38`.

**3. `quick_recovery_areas`.** Trigger: contains `"recovered quickly"`, `"quickly recovered"`,
or `"short duration"`. Ranks units **ascending** by duration, shortest flood spans first,
defaulting to buildings if no type is named:

```sparql
SELECT ?unit ?duration
WHERE {
  ?unit ex:type "Building" .
  {
    SELECT ?unit (MIN(?time) AS ?start) (MAX(?time) AS ?end)
    WHERE {
      ?unit ex:type "Building" .
      ?state ex:ofUnit ?unit ; ex:occursAt ?time ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
      FILTER(xsd:float(?depth) > 0.1)
    }
    GROUP BY ?unit
  }
  BIND(xsd:integer(REPLACE(STR(?end), ".*_(\\d+)$", "$1")) - xsd:integer(REPLACE(STR(?start), ".*_(\\d+)$", "$1")) AS ?duration)
}
ORDER BY ?duration
LIMIT 20
```

> NL question: *"Which areas recovered quickly?"*

This surfaces the buildings with the shortest flood spans first, in ascending order of
duration.

**4. `high_risk_check`**: documented under [`A-01`](#a-01); combines peak depth and flood
duration into a single risk check.

**5. `highest_risk_overall`.** Trigger: contains `"highest risk overall"` or `"max depth"`
**together with** `"critical infrastructure"`/`"critical infra"`/`"crit-infra"`. Ranks
hospitals, schools, and power facilities by peak depth:

```sparql
SELECT ?unit (MAX(xsd:float(?depth)) AS ?maxDepth)
WHERE {
  ?unit ex:type ?entityType .
  FILTER(?entityType IN ("Hospital", "PowerFacility", "School"))
  ?state ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
GROUP BY ?unit
ORDER BY DESC(?maxDepth)
LIMIT 20
```

> NL question: *"What is the highest risk overall for critical infrastructure?"*

Real-KG row: `ex:Hospital_0249c5e672 → 23.0`, ranked against every other hospital, school, and
power facility by peak depth.

**6. `longest_flooded`.** Trigger: contains `"flooded the longest"`. Ranks every unit by total
flooded timesteps:

```sparql
SELECT ?unit (COUNT(?state) AS ?floodDuration)
WHERE {
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
GROUP BY ?unit
ORDER BY DESC(?floodDuration)
LIMIT 20
```

> NL question: *"Which units flooded the longest?"*

Real-KG ranking: `RoadSegment_000219a31d (49)` > `Building_00111dfad8 (39)` >
`Hospital_0249c5e672 (38)`.

**7. `peak_flooding_units`.** Trigger: contains `"experienced peak flooding"` or
`"which units experienced peak flooding"`. Finds the single deepest flood depth recorded
anywhere in the simulation, and returns the unit(s) and time(s) where it occurred:

```sparql
SELECT ?unit ?time ?depth
WHERE {
  {
    SELECT (MAX(xsd:float(?d)) AS ?maxDepth)
    WHERE {
      ?o a ex:DepthObservation ;
         ex:depthValue ?d .
    }
  }
  ?state ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) = ?maxDepth)
}
LIMIT 10
```

> NL question: *"Which units experienced peak flooding?"*

Real-KG result: `(ex:Building_00111dfad8, <its peak instant>, 31.0)`, the highest depth
recorded among this library's tracked entities.

**8. `severe_overall`.** Trigger: contains `"severely affected overall"` or `"severe overall"`.
Ranks units whose peak depth exceeds `2.0` m:

```sparql
SELECT ?unit (MAX(xsd:float(?depth)) AS ?maxDepth)
WHERE {
  ?state ex:ofUnit ?unit ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
GROUP BY ?unit
HAVING(?maxDepth > 2.0)
ORDER BY DESC(?maxDepth)
LIMIT 10
```

> NL question: *"Which units were severely affected overall?"*

Real-KG ranking: `Building_00111dfad8 (31.0)` > `RoadSegment_000219a31d (30.0)` >
`Hospital_0249c5e672 (23.0)`. All three clear the `2.0` m bar.

---

## A-03

### How many units flooded now?

**Purpose.** Count distinct units currently exceeding the depth threshold at a given time.

**Logic / method:**
1. Bind the selected `TimeInstant`.
2. Inspect all candidate `FloodUnitState` records.
3. Keep units whose depth exceeds the threshold.
4. Count distinct flooded units.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `TIME_INSTANT` | Optional | `TimeInstant_011` |
| `DEPTH_THRESHOLD` | Optional | `0.1` |

**Rule-trigger phrase(s):**
- Contains `"how many"` **and** `"affected overall"`/`"affected in total"`/`"overall"` →
  `A-03 / affected_count_overall` (`0.98`).
- Contains `"how many"`/`"count"` **and** `"flooded"`/`"flooding"` →
  `A-03 / flooded_count` (`0.98`).

**SPARQL template**

```sparql
SELECT (COUNT(DISTINCT ?unit) AS ?floodedCount)
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
```

**Worked example**

> NL question: *"How many units are flooded right now?"*

`TIME_INSTANT` defaults to `TimeInstant_011`. The query counts every distinct unit with a
qualifying state at that instant, including `ex:RoadSegment_000219a31d`, which is flooded
starting at exactly `TimeInstant_011`.

#### Extended dynamic patterns

**1. `affected_count_overall`.** Counts units that have ever been the target of a
`FloodEvent`, regardless of time:

```sparql
SELECT (COUNT(DISTINCT ?unit) AS ?affectedUnits)
WHERE {
  ?unit ex:type "Building" .
  ?event ex:derivedFromState ?state .
  ?state ex:ofUnit ?unit .
}
```

> NL question: *"How many buildings were affected overall?"* (`ENTITY_TYPE=Building`), counting
> every building linked to a `FloodEvent` through its states.

**2. Typed snapshot count (`A-03` + `ENTITY_TYPE`).** Adds a type filter to the time-bound
count:

```sparql
SELECT (COUNT(DISTINCT ?unit) AS ?floodedCount)
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
```

> NL question: *"How many hospitals are flooded at TimeInstant_013?"*

Real-KG result includes `ex:Hospital_0249c5e672`, which is flooded at that instant.
