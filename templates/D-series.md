# D-series: Decision-Support Queries

> Operational evacuation and infrastructure-prioritization analytics.
> Defining example: *"Which buildings should be prioritized for evacuation?"*

This family has **1 static template** (`D-01`), combining two of the patterns established in
the other families, earliest flood onset (`T-01`'s logic) and peak depth (`A-01`'s logic),
into a single combined evacuation-priority ranking.

---

## D-01

### Evacuation priorities

**Purpose.** Rank buildings by evacuation urgency, combining **early flood onset** with
**high peak depth**, the two factors that make a building a priority to evacuate: it flooded
soon after the event began, and it flooded deeply.
(`human_intent`: *"Rank buildings that should receive evacuation or intervention attention
first."*)

**Logic / method:**
1. Retrieve building units.
2. Find each building's first flooded time.
3. Find each building's peak observed flood depth.
4. Rank buildings using onset and peak severity.
5. Return the top evacuation-priority candidates.

**Ontology elements touched:** `ex:SpatialUnit`, `ex:FloodUnitState`, `ex:DepthObservation` via
`ex:type`, `ex:ofUnit`, `ex:occursAt`, `ex:hasObservation`, `ex:depthValue`.

**Parameters**

| Parameter | Required? | Default |
| --- | --- | --- |
| `DEPTH_THRESHOLD` | Optional | `0.1` (used by the onset subquery to decide what counts as "flooded" when finding the earliest qualifying time) |
| `LIMIT_VALUE` | Optional | `20` |

**Rule-trigger phrase(s):** contains `"evacuate"`, `"evacuation"`, `"priority"`, or
`"priorities"` → `D-01 / evacuation_priority` (`0.99`).

**SPARQL template**

```sparql
SELECT ?unit ?start ?peak
WHERE {
  ?unit ex:type "Building" .
  {
    SELECT ?unit (MIN(?time) AS ?start)
    WHERE {
      ?state ex:ofUnit ?unit ; ex:occursAt ?time ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
      FILTER(xsd:float(?depth) > {DEPTH_THRESHOLD})
    }
    GROUP BY ?unit
  }
  {
    SELECT ?unit (MAX(xsd:float(?depth)) AS ?peak)
    WHERE {
      ?state ex:ofUnit ?unit ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
    }
    GROUP BY ?unit
  }
}
ORDER BY ?start DESC(?peak)
LIMIT {LIMIT_VALUE}
```

`ORDER BY ?start DESC(?peak)` ranks buildings primarily by ascending `start` time, so buildings
that flooded earliest, and therefore had the least warning time, are ranked first; for any
buildings that started flooding at exactly the same instant, the tie is broken in favor of the
one with the higher peak depth. This is the SPARQL expression of the "early onset + high peak"
ranking metric named in `explanation_builder.TEMPLATE_EXPLANATIONS["D-01"]`.

**Worked example: this category's defining example, verbatim**

> NL question: *"Which buildings should be prioritized for evacuation?"*

`DEPTH_THRESHOLD=0.1` and `LIMIT_VALUE=20` both fall back to their defaults.

```sparql
SELECT ?unit ?start ?peak
WHERE {
  ?unit ex:type "Building" .
  {
    SELECT ?unit (MIN(?time) AS ?start)
    WHERE {
      ?state ex:ofUnit ?unit ; ex:occursAt ?time ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
      FILTER(xsd:float(?depth) > 0.1)
    }
    GROUP BY ?unit
  }
  {
    SELECT ?unit (MAX(xsd:float(?depth)) AS ?peak)
    WHERE {
      ?state ex:ofUnit ?unit ; ex:hasObservation ?obs .
      ?obs a ex:DepthObservation ; ex:depthValue ?depth .
    }
    GROUP BY ?unit
  }
}
ORDER BY ?start DESC(?peak)
LIMIT 20
```

Result row: `(ex:Building_00111dfad8, ex:TimeInstant_016, 31.0)`. This building started
flooding at `TimeInstant_016` and reached a peak depth of `31.0` m, placing it among the
top evacuation-priority candidates returned by the ranking.
