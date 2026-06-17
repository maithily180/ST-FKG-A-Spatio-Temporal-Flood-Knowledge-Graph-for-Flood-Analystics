# E-series: Event Queries

> Flood-event occurrence and event-history analytics.
> Defining example: *"What flood events occurred at this hospital?"*

Event queries operate one layer up from raw state: instead of reading `FloodUnitState`/
`DepthObservation` directly, they read `ex:FloodEvent` nodes: the recognized `eventType`
values (`FloodStart`, `FloodEnd`, `PeakFlood`, `PersistentFlood`, `SeverityChange`,
`RapidRise`, `RapidFall`, `HighVelocity`, `CriticalInfrastructureImpact`, per `ONTOLOGY.md` §3)
derived from the raw flood states. There are **2 static templates** (`E-01`, `E-03`) and
**4 dynamic patterns**.

---

## E-01

### Did unit experience event?

**Purpose.** Check whether a unit was affected by a specific named event type (e.g. "did this
unit have a `RapidRise` event?"). (`human_intent`: *"Check whether a unit or group of units is
linked to a named flood-event type."*)

**Logic / method:**
1. Use the requested event type.
2. Find `FloodEvent` nodes with that `eventType`.
3. Follow impact links to affected units.
4. Return event counts or impacted units.

**Ontology elements touched:** `ex:SpatialUnit`, `ex:FloodEvent` via `ex:eventType`,
`ex:impacts`, `ex:occursAt`, `ex:derivedFromState`.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `UNIT_ID` | Optional | none |
| `EVENT_TYPE` | **Required** | none |

**Rule-trigger phrase(s), in priority order:**
1. A unit is named **and** the question contains `"multiple flood events"` or
   `"multiple event types"` → `E-01 / multiple_event_types_count` (`0.98`).
2. Plural scope **and** the question contains `"multiple event types"` →
   `E-01 / multiple_event_types_units` (`0.98`).
3. An `EVENT_TYPE` was extracted **and** the question contains `"which units"`,
   `"which buildings"`, `"which roads"`, `"which hospitals"`, `"list units"`, or
   `"show units"` → `E-01 / event_units` (`0.97`).
4. A unit is named **and** (the literal substring `" event"` appears, OR an `EVENT_TYPE` was
   extracted) → `E-01 / event_occurred` (`0.96`).

The `multiple_event_types_*` patterns only need `UNIT_ID`. They count *distinct* event types
rather than checking one named type.

**SPARQL template (`event_occurred`)**

```sparql
SELECT (COUNT(?event) AS ?eventCount)
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  BIND("{EVENT_TYPE}" AS ?eventType)
  ?event a ex:FloodEvent ;
         ex:eventType ?eventType ;
         ex:impacts ?unit .
}
```

**Worked example**

> NL question: *"Did RoadSegment_000219a31d experience FloodStart events?"*

`EVENT_TYPE=FloodStart` is extracted from the `"<Name> events"` pattern.

```sparql
SELECT (COUNT(?event) AS ?eventCount)
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?unit)
  BIND("FloodStart" AS ?eventType)
  ?event a ex:FloodEvent ;
         ex:eventType ?eventType ;
         ex:impacts ?unit .
}
```

Result: `eventCount = 1`, matching the real KG triple
`ex:FloodEvent_0000000 a ex:FloodEvent ; ex:eventType "FloodStart" ; ex:impacts
ex:RoadSegment_000219a31d ; ex:occursAt ex:TimeInstant_011 ; ex:eventID "EV_0000000" .`

#### Extended dynamic patterns

**1. `multiple_event_types_count`.** Trigger: a unit is named **and** the question contains
"multiple flood events" or "multiple event types". Counts how many distinct `eventType` values
are linked to one unit:

```sparql
SELECT (COUNT(DISTINCT ?type) AS ?eventTypes)
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?building)
  ?event a ex:FloodEvent ;
         ex:impacts ?building ;
         ex:eventType ?type .
}
```

> NL question: *"Did RoadSegment_000219a31d experience multiple flood events?"*

**2. `multiple_event_types_units`.** Trigger: plural scope **and** "multiple event types".
Ranks all units by their distinct-event-type count, restricted to units with more than 2:

```sparql
SELECT ?unit (COUNT(DISTINCT ?etype) AS ?eventTypes)
WHERE {
  ?event ex:derivedFromState ?state ;
         ex:eventType ?etype .
  ?state ex:ofUnit ?unit .
}
GROUP BY ?unit
HAVING(?eventTypes > 2)
ORDER BY DESC(?eventTypes)
LIMIT 10
```

> NL question: *"Which units have multiple event types?"*

**3. `event_units`.** Trigger: an `EVENT_TYPE` was extracted **and** the question contains a
"which units"-style phrase. Returns `(unit, time)` pairs for every unit linked to that event
type, following both the direct-impact and derived-state paths:

```sparql
SELECT DISTINCT ?unit ?time
WHERE {
  BIND("FloodStart" AS ?eventType)
  ?event a ex:FloodEvent ;
         ex:eventType ?eventType ;
         ex:occursAt ?time .
  {
    ?event ex:derivedFromState ?state .
    ?state ex:ofUnit ?unit .
  }
  UNION
  {
    ?event ex:impacts ?unit .
  }
  ?unit ex:type "RoadSegment" .
}
LIMIT 10
```

> NL question: *"Which roads experienced FloodStart events?"* → `EVENT_TYPE=FloodStart`,
> `ENTITY_TYPE=RoadSegment`.

Real-KG row: `(ex:RoadSegment_000219a31d, ex:TimeInstant_011)`.

---

## E-03

### What events at unit?

**Purpose.** List every event affecting a unit, in chronological order. This is the direct
answer to this category's defining example once a concrete unit is named.

**Logic / method:**
1. Bind the requested infrastructure unit.
2. Find `FloodEvent` nodes that impact the unit.
3. Return each event's type and occurrence time.
4. Order events by time.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `UNIT_ID` | **Required** | none |

**Rule-trigger phrase(s):**
- A unit is named **and** the question contains `"peak flooding occur"`,
  `"peak flooding occurred"`, or `"peak flood occur"` → `E-03 / peak_flood_event` (`0.98`).
- Contains `"what events"`, `"which events"`, `"events happened"`, `"what happened"`, or
  `"list events"` → `E-03 / what_events` (`0.98`).

**SPARQL template**

```sparql
SELECT DISTINCT ?event ?type ?time
WHERE {
  BIND(ex:{UNIT_ID} AS ?unit)
  ?event a ex:FloodEvent ;
         ex:impacts ?unit ;
         ex:eventType ?type ;
         ex:occursAt ?time .
}
ORDER BY ?time
```

**Worked example**

> NL question: *"What events happened at RoadSegment_000219a31d?"*

```sparql
SELECT DISTINCT ?event ?type ?time
WHERE {
  BIND(ex:RoadSegment_000219a31d AS ?unit)
  ?event a ex:FloodEvent ;
         ex:impacts ?unit ;
         ex:eventType ?type ;
         ex:occursAt ?time .
}
ORDER BY ?time
```

Result: `(ex:FloodEvent_0000000, "FloodStart", ex:TimeInstant_011)`, followed by the unit's
later events in chronological order.

#### Extended dynamic pattern: `peak_flood_event`

**Trigger:** a unit is named **and** the question contains "peak flooding occur(red)" or
"peak flood occur".

```sparql
SELECT ?event ?time
WHERE {
  BIND(ex:{UNIT_ID} AS ?building)
  ?event a ex:FloodEvent ;
         ex:eventType "PeakFlood" ;
         ex:impacts ?building ;
         ex:occursAt ?time .
}
```

> NL question: *"When did peak flooding occur at RoadSegment_000219a31d?"*

This returns the `time` of the unit's `PeakFlood`-typed event, the moment the pipeline marked
as the unit's flood crest.
