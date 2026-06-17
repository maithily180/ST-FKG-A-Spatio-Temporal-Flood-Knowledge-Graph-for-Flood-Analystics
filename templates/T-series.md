# T-series: Temporal Queries

> Flood arrival, duration, persistence, and temporal flood-evolution analysis.
> Defining example: *"When did flooding first reach a hospital?"*

Temporal queries trace a unit's relationship to time: when it first flooded, when it stopped,
how long it stayed flooded, and how its depth evolved over the full simulation window. There
are **4 static templates** (`T-01`–`T-04`) and **6 dynamic patterns**.

---

## T-01

### When did flooding start?

**Purpose.** Find when a unit first exceeded the depth threshold: flood arrival time.
(`human_intent`: *"Find when flooding first reached the selected infrastructure unit."*)

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Scan all time-indexed `FloodUnitState` records.
3. Keep only states where depth exceeds the threshold.
4. Return the minimum `TimeInstant` as flood arrival.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `UNIT_ID` | **Required** | none |
| `DEPTH_THRESHOLD` | Optional | `0.1` |

**Rule-trigger phrase(s):** `"when"` **and** one of `"start"`, `"began"`, `"begin"`,
`"impassable"`, `"become affected"`, `"became affected"` → `T-01 / flood_start` (`0.99`).

**SPARQL template**

```sparql
SELECT (MIN(?time) AS ?startTime)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
}
```

**Worked example**

> NL question: *"When did flooding start at Hospital_0249c5e672?"*

```sparql
SELECT (MIN(?time) AS ?startTime)
WHERE {
  BIND(ex:Hospital_0249c5e672 AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
```

Result: `startTime = ex:TimeInstant_013`. This is this hospital's flood arrival time, directly
answering this category's defining example.

#### Extended dynamic patterns

**1. `flood_start_by_unit` (plural scope).** Trigger: plural scope (e.g. a bare plural like
"hospitals"), `"when"` **and** a start-keyword. Reports the arrival time for every unit of a
type at once:

```sparql
SELECT ?unit (MIN(?time) AS ?startTime)
WHERE {
  ?unit ex:type "Hospital" .
  ?state ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
GROUP BY ?unit
ORDER BY ?startTime
LIMIT 10
```

> NL question: *"When did flooding start at the hospitals?"*

Real-KG row: `ex:Hospital_0249c5e672 → TimeInstant_013`, ranked alongside other hospitals by
earliest start time.

**2. `earlier_between_units`.** Trigger: a second unit is named **and** the question contains
"flooded earlier". Compares the arrival time of two named units directly:

```sparql
SELECT ?which (MIN(?t) AS ?earliest)
WHERE {
  VALUES (?which ?building) {
    ("A" ex:RoadSegment_000219a31d)
    ("B" ex:Building_00111dfad8)
  }
  ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?t ; ex:hasObservation ?o .
  ?o a ex:DepthObservation ; ex:depthValue ?d .
  FILTER(xsd:float(?d) > 0.1)
}
GROUP BY ?which
ORDER BY ?earliest
```

> NL question: *"Which flooded earlier, RoadSegment_000219a31d or Building_00111dfad8?"*

Real-KG result: `A (RoadSegment_000219a31d) → TimeInstant_011`,
`B (Building_00111dfad8) → TimeInstant_016`. The road segment flooded earlier.

---

## T-02

### When did flooding end?

**Purpose.** Find when a unit last exceeded the depth threshold: flood recession time.

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Scan all flooded time-indexed states.
3. Keep states with depth above the threshold.
4. Return the maximum `TimeInstant` as recession/end time.

**Parameters:** same shape as `T-01` (`UNIT_ID` required, `DEPTH_THRESHOLD` optional, default
`0.1`).

**Rule-trigger phrase(s):** `"when"` **and** one of `"stop"`, `"stopped"`, `"end"`, `"ended"` →
`T-02 / flood_end` (`0.99`).

**SPARQL template**

```sparql
SELECT (MAX(?time) AS ?endTime)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
}
```

**Worked example**

> NL question: *"When did flooding end at Building_00111dfad8?"*

```sparql
SELECT (MAX(?time) AS ?endTime)
WHERE {
  BIND(ex:Building_00111dfad8 AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
```

Result: `endTime = ex:TimeInstant_054`. This is this building's flood recession time.

#### Extended dynamic pattern: `flood_end_by_unit` (plural scope)

Trigger: plural scope, `"when"` **and** a stop/end-keyword. Reports the recession time for
every unit of a type at once:

```sparql
SELECT ?unit (MAX(?time) AS ?endTime)
WHERE {
  ?unit ex:type "Building" .
  ?state ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
  FILTER(xsd:float(?depth) > 0.1)
}
GROUP BY ?unit
ORDER BY ?endTime
LIMIT 10
```

> NL question: *"When did flooding end at the buildings?"*

Real-KG row: `ex:Building_00111dfad8 → TimeInstant_054`, ranked alongside other buildings by
end time.

---

## T-03

### How long was unit flooded?

**Purpose.** Calculate flood duration in timesteps, the difference between a unit's last and
first flooded instants.

**Logic / method:**
1. Find the first flooded `TimeInstant` (nested `MIN` subquery).
2. Find the last flooded `TimeInstant` (nested `MAX` subquery).
3. Convert both time identifiers to integer timestep numbers via regex `REPLACE` on the local
   name (`TimeInstant_011` → `11`).
4. Subtract start from end to obtain flood duration.

**Parameters:** `UNIT_ID` required, `DEPTH_THRESHOLD` optional (default `0.1`).

**Rule-trigger phrase(s):**
- Contains `"how long"` or `"duration"` → `T-03 / flood_duration` (`0.99`).
- A unit is named, the question contains `"flooded"`, **no** time instant is present, and it
  does **not** contain `"when"`/`"which"`/`"how many"`/`"safe"`/`"depth"` →
  `T-03 / flood_duration` (`0.90`). This covers a bare statement like
  "Was Building X flooded?" used as a duration question.

**SPARQL template**

```sparql
SELECT (?endTimeNum - ?startTimeNum AS ?duration)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  {
    SELECT (MIN(?time) AS ?startTime)
    WHERE {
      ?state a ex:FloodUnitState ; ex:ofUnit ?unit ; ex:occursAt ?time ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
      FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
    }
  }
  {
    SELECT (MAX(?time) AS ?endTime)
    WHERE {
      ?state a ex:FloodUnitState ; ex:ofUnit ?unit ; ex:occursAt ?time ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
      FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
    }
  }
  BIND(xsd:integer(REPLACE(STR(?startTime), ".*_(\\d+)$", "$1")) AS ?startTimeNum)
  BIND(xsd:integer(REPLACE(STR(?endTime), ".*_(\\d+)$", "$1")) AS ?endTimeNum)
}
```

**Worked example**

> NL question: *"How long was RoadSegment_000219a31d flooded?"*

Substituting `UNIT_ID=RoadSegment_000219a31d`, `DEPTH_THRESHOLD=0.1`: the nested subqueries
resolve to `startTime=TimeInstant_011`, `endTime=TimeInstant_059`, so
`startTimeNum=11`, `endTimeNum=59`.

Result: `duration = 48` (timesteps).

Second example: *"How long was Building_00111dfad8 flooded?"* → `startTime=TimeInstant_016`,
`endTime=TimeInstant_054` → `duration = 38`.

#### Extended dynamic pattern: `disconnected_intervals`

**Trigger:** the question contains `"dry and flood again"`, `"flood again"`, or
`"disconnected interval"`.

This estimates how many distinct ~10-timestep windows of the simulation a unit's flooding
touched, by bucketing each flooded timestep into a group of 10 (`FLOOR(timestep / 10)`) and
counting the distinct groups reached:

```sparql
SELECT (COUNT(DISTINCT ?gap) AS ?intervalCount)
WHERE {
  BIND(ex:{UNIT_ID} AS ?building)
  ?state a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?time ; ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ; ex:depthValue ?depth . FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
  BIND(xsd:integer(REPLACE(STR(?time), ".*_(\\d+)$", "$1")) AS ?tidx)
  BIND(FLOOR(?tidx / 10) AS ?gap)
}
```

> NL question: *"Did Hospital_0249c5e672 dry out and flood again?"*

Real-KG result: flooded from `T013` through `T050`, spanning buckets `1` (`T013`–`T019`)
through `5` (`T050`–`T059`) → `intervalCount = 5`.

---

## T-04

### Full flood timeline

**Purpose.** Get the complete time-series of depth values for a unit. This is the richest
single-unit temporal view, and the basis for the "flood evolution" framing in the category
description.

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Retrieve all linked `FloodUnitState` records.
3. Read depth observations for each `TimeInstant`.
4. Sort the sequence by time.

**Parameters:** `UNIT_ID` required; no thresholds, since this template is unfiltered and
returns every recorded depth regardless of magnitude.

**Rule-trigger phrase(s):** a unit is named **and** (contains "timeline"/"history"/"over
time"/"all depths"/"flood story", OR contains "tell me about" together with "flood"/"flooding",
OR contains "situation") → `T-04 / flood_timeline` (`0.95`).

**SPARQL template**

```sparql
SELECT ?time ?depth
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
ORDER BY ?time
```

**Worked example**

> NL question: *"Show me the flood timeline for RoadSegment_000219a31d."*

```sparql
SELECT ?time ?depth
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?unit)
  ?state a ex:FloodUnitState ;
         ex:ofUnit ?unit ;
         ex:occursAt ?time ;
         ex:hasObservation ?obs .
  ?obs a ex:DepthObservation ;
       ex:depthValue ?depth .
}
ORDER BY ?time
```

Result: a 49-row sequence from `(TimeInstant_011, depth₀)` through `(TimeInstant_059,
depth₄₈)`, rising to a peak of `30.0` m (Severe) before receding. This is the same data `A-01`
reduces to a single `MAX`, here returned in full.

#### Extended dynamic patterns

**1. `full_flood_story`.** Trigger: a unit is named **and** the question contains "full flood
story" or "flood story". Pre-aggregates start, peak (time + depth), end, and the unit's full
event list in one query:

```sparql
SELECT ?start ?peakTime ?peakDepth ?end ?events
WHERE {
  BIND(ex:{UNIT_ID} AS ?building)
  {
    SELECT (MIN(?time) AS ?start)
    WHERE {
      ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?time ; ex:hasObservation ?o .
      ?o a ex:DepthObservation ; ex:depthValue ?d . FILTER(xsd:float(?d) > {DEPTH_THRESHOLD})
    }
  }
  {
    SELECT (MAX(xsd:float(?depth)) AS ?peakDepth) (MIN(?time) AS ?peakTime)
    WHERE {
      ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?time ; ex:hasObservation ?o .
      ?o a ex:DepthObservation ; ex:depthValue ?depth .
    }
  }
  {
    SELECT (MAX(?time) AS ?end)
    WHERE {
      ?s a ex:FloodUnitState ; ex:ofUnit ?building ; ex:occursAt ?time ; ex:hasObservation ?o .
      ?o a ex:DepthObservation ; ex:depthValue ?d . FILTER(xsd:float(?d) > {DEPTH_THRESHOLD})
    }
  }
  {
    SELECT (GROUP_CONCAT(DISTINCT ?tname; separator=", ") AS ?events)
    WHERE {
      ?event a ex:FloodEvent ; ex:impacts ?building ; ex:eventType ?type .
      BIND(CONCAT(STR(?type), "@", STR(?event)) AS ?tname)
    }
  }
}
```

> NL question: *"Tell me the full flood story for Building_00111dfad8."*

Result: `start = TimeInstant_016`, `end = TimeInstant_054`, `peakDepth = 31.0`, `peakTime` is
the instant that first records that maximum, and `events` is a comma-joined list of every
`FloodEvent` impacting the building, formatted as `"<eventType>@<eventURI>"`. For example,
the road segment's analogous event renders as
`"FloodStart@http://www.semanticweb.org/.../FloodEvent_0000000"`.

**2. `compare_flooding_units`.** Trigger: a second unit is named **and** the question contains
"compare flooding". Compares the peak flood depth reached by each of the two named units:

```sparql
SELECT ?metric ?A ?B
WHERE {
  BIND(ex:{UNIT_ID} AS ?Aunit)
  BIND(ex:{SECOND_UNIT_ID} AS ?Bunit)
  { SELECT ("A" AS ?metric) (MAX(xsd:float(?depth)) AS ?A)
    WHERE {
      ?state ex:ofUnit ?Aunit ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
    }
  }
  { SELECT ("B" AS ?metric) (MAX(xsd:float(?depth)) AS ?B)
    WHERE {
      ?state ex:ofUnit ?Bunit ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
    }
  }
}
```

> NL question: *"Compare flooding between RoadSegment_000219a31d and Building_00111dfad8."*

Real-KG result: `A (RoadSegment_000219a31d) = 30.0`, `B (Building_00111dfad8) = 31.0`. The
building's peak was slightly deeper than the road's.
