# Type Composition

## Intersections and Unions

An intersection is a subtype of every member:

```xeto
Named : Dict { dis: Str }
Located : Dict { geoCity: Str }
NamedLocation : Named & Located
```

### Vocabulary

```turtle
ex:Named a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict .

ex:Named.dis a rdf:Property ;
  rdfs:label "dis"@en .

ex:Located a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict .

ex:Located.geoCity a rdf:Property ;
  rdfs:label "geoCity"@en .

ex:NamedLocation a sys:Class, rdfs:Class ;
  rdfs:subClassOf ex:Named, ex:Located ;
  rdfs:label "NamedLocation"@en .
```

The inherited slots keep the property IRIs of the specs that declared them:
`ex:Named.dis` and `ex:Located.geoCity`. They are not copied to new properties
such as `ex:NamedLocation.dis`.

A union is a supertype of every member:

```xeto
Heating : Dict
Cooling : Dict
ThermalProcess : Heating | Cooling
```

```turtle
ex:ThermalProcess a sys:Class, rdfs:Class .

ex:Heating a sys:Class, rdfs:Class ;
  rdfs:subClassOf ex:ThermalProcess .

ex:Cooling a sys:Class, rdfs:Class ;
  rdfs:subClassOf ex:ThermalProcess .
```

### SHACL Validation

**Intersection.** The member shapes define the constraints inherited by an
intersection:

```turtle
ex:Named a sh:NodeShape ;
  sh:targetClass ex:Named ;
  sh:property [
    sh:path ex:Named.dis ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .

ex:Located a sh:NodeShape ;
  sh:targetClass ex:Located ;
  sh:property [
    sh:path ex:Located.geoCity ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

Because `NamedLocation` is a subtype of both `Named` and `Located`, its
instances must satisfy both shapes.

**Union.** A union shape requires an instance to use at least one concrete
member class:

```turtle
ex:ThermalProcess a sh:NodeShape ;
  sh:targetClass ex:ThermalProcess ;
  sh:or (
    [ sh:class ex:Heating ]
    [ sh:class ex:Cooling ]
  ) .
```

This Xeto instance selects the concrete `Heating` member:

```xeto
@heating1: Heating {}
```

Its RDF representation conforms to the union shape because its type is one of
the member classes:

```turtle
ex:heating1 a sys:Entity, ex:Heating .
```

The union type itself cannot be used as the instance's concrete type:

```xeto
// Invalid: no concrete union member is selected.
@process1: ThermalProcess {}
```

If represented in RDF, that invalid instance names only the union class:

```turtle
ex:process1 a sys:Entity, ex:ThermalProcess .
```

That RDF fails `sh:or` because it has neither `rdf:type ex:Heating` nor
`rdf:type ex:Cooling`. Adding fields associated with a member does not select
that member; the instance must use the member spec as its concrete Xeto type.
The `rdfs:subClassOf` statements still make valid `Heating` and `Cooling`
instances members of `ThermalProcess`.

## Globals and Covariant Overrides

A global slot defines a contract for uses of the same slot name on descendant
specs. A global is implicitly optional and cannot declare `maybe`. Any
descendant that declares or supplies the slot must preserve the global
contract:

```xeto
Person : Dict {
  *height: Number <minVal:0, maxVal:300>
}

TallPerson : Person {
  height: Int <minVal:100>
}
```

`TallPerson.height` is a valid narrowing: `Int` is narrower than `Number`,
`minVal:100` raises the inherited minimum of zero, and the inherited maximum
remains 300.

This override instead widens the value space to an unrelated type, so Xeto
rejects it before RDF is emitted:

```xeto
InvalidPerson : Person {
  height: Str
}
```

### Vocabulary

A concrete slot that overrides a known global is an RDF subproperty of the
global slot property:

```turtle
ex:Person.height a rdf:Property ;
  rdfs:label "height"@en .

ex:TallPerson.height a rdf:Property ;
  rdfs:label "height"@en ;
  rdfs:subPropertyOf ex:Person.height .
```

The subproperty statement records where the concrete slot contract came from.
It does not by itself enforce the inherited datatype or numeric bounds.

### SHACL Validation

The effective `TallPerson.height` shape combines the override with inherited
global metadata:

```turtle
ex:TallPerson a sh:NodeShape ;
  sh:targetClass ex:TallPerson ;
  sh:property [
    sh:path ex:TallPerson.height ;
    sh:datatype xsd:integer ;
    sh:minInclusive 100 ;
    sh:maxInclusive 300 ;
    sh:maxCount 1 ;
  ] .
```

The shape uses `xsd:integer` and `sh:minInclusive 100` from the narrower
override, while `sh:maxInclusive 300` and the optional cardinality come from
the global contract. Xeto requires an overriding slot to narrow, rather than
widen, the inherited slot. The exporter checks that rule before producing RDF
and rejects an invalid override. SHACL then validates instance data against the
accepted effective slot. SHACL is not used to decide whether a Xeto slot
override only narrows the slot it overrides.

### Instance Data

Global-backed values use the concrete slot property of the instance's spec:

```xeto
@tall1: TallPerson {
  height: 175
}
```

```turtle
ex:tall1 a sys:Entity, ex:TallPerson ;
  ex:TallPerson.height "175"^^xsd:integer .
```

The same RDF structure with a value above 300 fails the inherited
`sh:maxInclusive` constraint. Omitting `height` remains valid because the
global declared it as `maybe`.

## Sealed Specs

`sealed` controls where a Xeto spec may be extended. A sealed spec may have
subtypes in its own library but not in another library. The exporter receives a
namespace only after Xeto has enforced this rule, so `sealed` does not add a
SHACL instance constraint or require an RDF vocabulary statement.

```xeto
Command : Dict <sealed>
```

An exporter reports an error if its input namespace contains an external
subtype of `Command`; it does not emit a weaker graph for that invalid schema.

## Mixins

A mixin adds slots to an existing spec in the active namespace. It does not
become a second RDF class. The target spec receives the mixin constraints, while
each contributed slot keeps the IRI of the library that contributed it.

`base.people` defines:

```xeto
Person : Dict {
  dis: Str
}
```

`app.people` defines:

```xeto
Org : Dict

+Person {
  badge: Str?
  staff
  orgRef: Ref<of:Org>
}
```

### Vocabulary

```turtle
app:Org a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "Org"@en .

app:Person.badge a rdf:Property ;
  rdfs:label "badge"@en .

app:Person.staff a sys:Marker ;
  rdfs:label "staff"@en .

base:Person sys:hasMarker app:Person.staff .

app:Person.orgRef a rdf:Property ;
  rdfs:label "orgRef"@en .
```

There is no `app:Person` class. The `app:Person.badge`, `app:Person.staff`, and
`app:Person.orgRef` IRIs identify the scalar, marker, and reference slots
contributed by `app.people`.

### SHACL Validation

In a namespace containing both libraries, the mixin constraints are added to
the effective target shape:

```turtle
base:Person a sh:NodeShape ;
  sh:targetClass base:Person ;
  sh:property [
    sh:path base:Person.dis ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path app:Person.badge ;
    sh:datatype xsd:string ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:hasValue app:Person.staff ;
  ] ;
  sh:property [
    sh:path app:Person.orgRef ;
    sh:class app:Org ;
    sh:nodeKind sh:IRI ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

The subject being validated remains a `base:Person`. Other supported slot forms
contributed by a mixin use their ordinary validation mappings on the target
shape.

Mixin query-path ownership is defined under
[Queries Contributed by Mixins](QuerySlots.md#queries-contributed-by-mixins).

### Instance Data

An instance uses the target spec's class and the contributed slots' own IRIs:

```xeto
@o1: Org {}

@p1: base.people::Person {
  dis: "Ann"
  badge: "A-17"
  staff
  orgRef: @o1
}
```

```turtle
app:o1 a sys:Entity, app:Org .

app:p1 a sys:Entity, base:Person ;
  base:Person.dis "Ann"^^xsd:string ;
  app:Person.badge "A-17"^^xsd:string ;
  sys:hasMarker app:Person.staff ;
  app:Person.orgRef app:o1 .
```

