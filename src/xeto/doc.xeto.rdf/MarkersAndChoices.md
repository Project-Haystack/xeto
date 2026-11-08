# Markers and Choices

## Markers

### Vocabulary

A marker slot is represented as a named marker resource rather than as a
boolean-valued property.

```xeto
Equip : Dict {
  equip
}
```

```turtle
ex:Equip sys:hasMarker ex:Equip.equip .

ex:Equip.equip a sys:Marker ;
  rdfs:label "equip"@en .
```

The first statement records that `equip` is a marker declared by `Equip`. The
instance-data mapping uses the same marker IRI when an instance carries that
marker.

### SHACL Validation

A required marker uses `sh:hasValue` on `sys:hasMarker`. It does not use
`sh:maxCount`, because one instance may carry many different markers.

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:hasValue ex:Equip.equip ;
  ] .
```

A `maybe` marker still has vocabulary metadata, but it does not add a required
`sh:hasValue` constraint. For example:

```xeto
Asset : Dict {
  temporary: Marker?
}
```

```turtle
ex:Asset sys:hasMarker ex:Asset.temporary .

ex:Asset.temporary a sys:Marker ;
  rdfs:label "temporary"@en .

ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset .
```

The marker is part of the exported vocabulary, but an `Asset` instance may
omit it because the shape has no `sh:hasValue ex:Asset.temporary` constraint.

### Instance Data

An instance marker uses `sys:hasMarker` and the marker resource declared by the
vocabulary mapping above.

```xeto
@equip1: Equip {
  equip
}
```

```turtle
ex:equip1 sys:hasMarker ex:Equip.equip .
```

Xeto may supply a required marker from the spec even when it is not repeated in
the authored instance. For example:

```xeto
Equip : Dict {
  equip
}

@equip1: Equip {}
```

The `equip` marker is required by `Equip`, so the exported instance includes
it once:

```turtle
ex:equip1 a sys:Entity, ex:Equip ;
  sys:hasMarker ex:Equip.equip .
```

## Choices

### Vocabulary

A choice root is a class, and every choice member is its subclass. Each member
is connected to the marker that selects it in Xeto.

```xeto
HeatingProcess : Choice
HotWaterHeating : HeatingProcess { hotWaterHeating }
SteamHeating : HeatingProcess { steamHeating }
```

```turtle
ex:HeatingProcess a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Choice ;
  rdfs:label "HeatingProcess"@en .

ex:HotWaterHeating a sys:Class, rdfs:Class ;
  rdfs:subClassOf ex:HeatingProcess ;
  rdfs:label "HotWaterHeating"@en ;
  sys:hasMarker ex:HotWaterHeating.hotWaterHeating .

ex:HotWaterHeating.hotWaterHeating a sys:Marker ;
  rdfs:label "hotWaterHeating"@en .

ex:SteamHeating a sys:Class, rdfs:Class ;
  rdfs:subClassOf ex:HeatingProcess ;
  rdfs:label "SteamHeating"@en ;
  sys:hasMarker ex:SteamHeating.steamHeating .

ex:SteamHeating.steamHeating a sys:Marker ;
  rdfs:label "steamHeating"@en .
```

Choice members are collected from the active namespace. A library may add a
member to a non-sealed choice defined by another library, so two namespaces can
produce different allowed-member lists for the same choice root.

Members may form more than one level of taxonomy. Every marker-bearing
descendant is a member of each Choice above it, not only of its immediate
parent.

```xeto
PointFunction : Choice
SensorPointFunction : PointFunction { sensor }
SyntheticPointFunction : PointFunction { synthetic }
ComputedSyntheticPointFunction : SyntheticPointFunction { computed }

Point : Dict {
  pointFunction: PointFunction?
}
```

The `Point.pointFunction` shape contains all three marker-bearing descendants:

```turtle
ex:Point a sh:NodeShape ;
  sh:targetClass ex:Point ;
  sh:property [
    sh:path ex:Point.pointFunction ;
    sh:in (
      ex:ComputedSyntheticPointFunction
      ex:SensorPointFunction
      ex:SyntheticPointFunction
    ) ;
    sh:maxCount 1 ;
  ] .
```

The exporter removes duplicate qualified names and orders the remaining names
consistently. The same rule applies when the Choice root or any descendants
come from loaded dependency libraries.

A slot may also use a marker-bearing intermediate Choice as its type. That
slot accepts the intermediate member itself and the marker-bearing descendants
under that branch. Members of sibling branches are not included in its
`sh:in` list.

### SHACL Validation

The vocabulary mapping above defines the choice root, member classes, and
member-marker metadata. The validation mapping defines how a slot selects one
or more of those members. A choice slot uses its ordinary slot property as an
RDF-native relationship to the selected member. `sh:in` checks exact member
IRIs; it does not treat the selected member as a nested object.

```xeto
Equip : Dict {
  heatingProcess: HeatingProcess
}
```

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    sh:path ex:Equip.heatingProcess ;
    sh:in (ex:HotWaterHeating ex:SteamHeating) ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

Choice cardinality follows the call-site metadata:

| Xeto choice slot | SHACL cardinality |
| --- | --- |
| `heatingProcess: HeatingProcess` | `sh:minCount 1`; `sh:maxCount 1` |
| `heatingProcess: HeatingProcess?` | no minimum; `sh:maxCount 1` |
| `heatingProcess: HeatingProcess <multiChoice>` | `sh:minCount 1`; no maximum |
| `heatingProcess: HeatingProcess? <multiChoice>` | no minimum or maximum |

Every value, including each value of a multi-choice slot, must appear in the
`sh:in` list. The list contains all members visible in the active namespace.

### Instance Data

The selected choice member is the object of the choice slot property.

```xeto
@equip1: Equip {
  hotWaterHeating
}
```

```turtle
ex:equip1 ex:Equip.heatingProcess ex:HotWaterHeating .
```

For multi-choice, the subject has one property value for each selected member.
For example:

```xeto
MultiFuelEquip : Dict {
  heatingProcesses: HeatingProcess <multiChoice>
}

@boiler1: MultiFuelEquip {
  hotWaterHeating
  steamHeating
}
```

```turtle
ex:boiler1 a sys:Entity, ex:MultiFuelEquip ;
  ex:MultiFuelEquip.heatingProcesses ex:HotWaterHeating,
                                      ex:SteamHeating .
```

The instance property points to the choice member IRI. The vocabulary graph
connects that member to the Xeto marker that selects it:

```turtle
ex:equip1 ex:Equip.heatingProcess ex:HotWaterHeating .
ex:HotWaterHeating sys:hasMarker ex:HotWaterHeating.hotWaterHeating .
```

A user can therefore find equipment selected by the `hotWaterHeating` marker
without requiring a second marker statement on every instance:

```sparql
SELECT ?equip WHERE {
  ?equip ex:Equip.heatingProcess ?choice .
  ?choice sys:hasMarker ex:HotWaterHeating.hotWaterHeating .
}
```

The exporter checks the completed instance's marker values, not marker names
found only on the spec. One matching member produces one choice-property value;
`multiChoice` produces one value for every matching member. The RDF instance
does not repeat those selection markers as `sys:hasMarker` statements. Other
instance markers are still exported normally.

