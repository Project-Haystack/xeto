<!--
title:      Mapping Rules
-->

This chapter describes the shared mappings for specs, slots, and instances.
The [mapping chapters](doc.xeto.rdf::index) group each construct's RDF vocabulary, SHACL
validation, and instance-data mappings. A subsection is omitted when the
construct has no separate mapping for that profile.

# Specs, Slots, and Instances

A single Xeto spec can supply RDF classes, SHACL shapes, and the properties used on instances.

## Vocabulary

Every Xeto spec is both a Xeto class and an RDFS class. Its name becomes an
English label, and its `doc` text becomes an English comment when present.

```xeto
Person : Dict <doc:"A person"> {
  dis: Str
}

Employee : Person
```

```turtle
ex:Person a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "Person"@en ;
  rdfs:comment "A person"@en .

ex:Employee a sys:Class, rdfs:Class ;
  rdfs:subClassOf ex:Person ;
  rdfs:label "Employee"@en .
```

The `rdfs:subClassOf` direction follows Xeto inheritance: every `Employee` is a
`Person`. References to specs in another library use that library's versioned
RDF namespace.

The Haxall exporter also writes two statements about the top of the hierarchy:

- `sys:Class rdfs:subClassOf rdfs:Class` identifies the class of exported Xeto
  specs. Haxall declares `sys:Class` as both an `rdfs:Class` and a `sh:NodeShape`.
- `sys:Obj rdfs:subClassOf rdfs:Resource` gives the root Xeto type an RDF
  superclass. `Obj` has no Xeto base type.

A **value-bearing slot** is a slot whose instance value is represented as the
object of an RDF statement, such as a string, number, reference, nested value,
choice, or list. This distinguishes it from a marker, which uses
`xetoRdf:hasMarker`, and a computed `Query`, which may use a property path.

Every value-bearing slot has its own property IRI. The local part combines the
spec name and slot name so slots with the same short name remain distinct.

```xeto
Site : Dict {
  site
}

Equip : Dict {
  dis: Str <doc:"Display name">
  siteRef: Ref<of:Site>
}
```

```turtle
ex:Site a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "Site"@en ;
  xetoRdf:hasMarker ex:Site.site .

ex:Site.site a sys:Marker ;
  rdfs:label "site"@en .

ex:Equip a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "Equip"@en .

ex:Equip.dis a rdf:Property ;
  rdfs:label "dis"@en ;
  rdfs:comment "Display name"@en .

ex:Equip.siteRef a rdf:Property ;
  rdfs:label "siteRef"@en .
```

The output preserves both classes and both slot properties. Property
declarations do not add an RDF domain or range: the
[SHACL Ref mapping](References.md) states that `ex:Equip.siteRef`
must point to an `ex:Site`, while the class declarations identify `Equip` and
`Site` themselves.

## SHACL Validation

A spec with instance constraints has a SHACL shape that targets its RDF class.

Vocabulary relationships determine which shapes apply. For example, a subtype
inherits its supertypes' constraints, an intersection is a subtype of every
member, and mixin properties are attached directly to the target shape. The corresponding RDF statements are shown under
[Intersections](TypeComposition.md#intersections), [Unions](TypeComposition.md#unions), and [Mixins](TypeComposition.md#mixins).

Ordinary Xeto slots use these cardinalities:

| Xeto slot | SHACL cardinality |
| --- | --- |
| required, single-valued | `sh:minCount 1`; `sh:maxCount 1` |
| `maybe`, single-valued | no minimum; `sh:maxCount 1` |
| required `MultiRef` | `sh:minCount 1`; no maximum |
| `maybe MultiRef` | no minimum or maximum |
| `Query` | no minimum or maximum |

The type and value constraints still apply when a `maybe` slot is present.
Additional cardinality rules apply to [markers](MarkersAndChoices.md#markers),
[choices](MarkersAndChoices.md#choices), and [lists](Lists.md).

### Reusable Field Shapes

An inline Dict gets its field checks from the slot that contains it. A
lookup record gets its field checks from its declared type.

```xeto
Address: Dict { city: Str }
SpecialAddress: Address { note: Str }
Person: Dict { address: Address }
```

The `person` record contains an inline address. The `home` record is a
separate address available for lookup.

```trio
id: @person
spec: @example::Person
address: {spec:@example::SpecialAddress city:"Here"}
---
id: @home
spec: @example::SpecialAddress
city: "Here"
```

Both addresses declare `SpecialAddress` and omit `note`. Xeto uses the
containing slot's type to check `person.address`. The slot asks for an
`Address`, and `SpecialAddress` fits because it's a subtype. The inline value
has the required `city`, so it passes. For `home`, Xeto checks the record
against its declared type, `SpecialAddress`. That type requires `note`, so
`home` fails.

In RDF, both addresses have type `ex:SpecialAddress`.
A bare `sh:targetClass ex:SpecialAddress` shape with a required `note` check
would reject both. The `sh:or` lets the inline address pass the class-targeted
shape so its containing slot can apply `ex:Address-fields`. The class-targeted
shape applies `ex:SpecialAddress-fields` to `home`.

The schema excerpt shows the class and field shapes for all three specs.

```turtle
ex:Address a rdfs:Class, sh:NodeShape ;
  rdfs:subClassOf sys:Dict ;
  sh:targetClass ex:Address ;
  sh:or (
    # Inline values pass this branch. The containing slot checks their fields.
    [ sh:not [ sh:class xetoRdf:Record ] ]
    [ sh:node ex:Address-fields ]
  ) .

ex:Address-fields a sh:NodeShape ;
  sh:property [
    sh:path ex:Address.city ;
    sh:datatype xsd:string ;
    sh:minCount 1 ; sh:maxCount 1
  ] .

ex:Person a rdfs:Class, sh:NodeShape ;
  rdfs:subClassOf sys:Dict ;
  sh:targetClass ex:Person ;
  sh:or (
    [ sh:not [ sh:class xetoRdf:Record ] ]
    # Check the fields of the lookup Person record.
    [ sh:node ex:Person-fields ]
  ) .

ex:Person-fields a sh:NodeShape ;
  sh:property [
    # Read the address stored on the Person record.
    sh:path ex:Person.address ;
    sh:nodeKind sh:BlankNode ;
    sh:or (
      [ sh:property [ sh:path sys:spec ; sh:maxCount 0 ] ]
      [ sh:class ex:Address ]
    ) ;
    # Check for the required city on the address value.
    sh:node ex:Address-fields ;
    sh:minCount 1 ; sh:maxCount 1
  ] .

ex:SpecialAddress a rdfs:Class, sh:NodeShape ;
  rdfs:subClassOf ex:Address ;
  sh:targetClass ex:SpecialAddress ;
  sh:or (
    # The inline address passes this branch.
    [ sh:not [ sh:class xetoRdf:Record ] ]
    # The lookup record must satisfy SpecialAddress's field requirements.
    [ sh:node ex:SpecialAddress-fields ]
  ) .

ex:SpecialAddress-fields a sh:NodeShape ;
  sh:property [
    sh:path ex:Address.city ;
    sh:datatype xsd:string ;
    sh:minCount 1 ; sh:maxCount 1
  ] ;
  sh:property [
    sh:path ex:SpecialAddress.note ;
    sh:datatype xsd:string ;
    sh:minCount 1 ; sh:maxCount 1
  ] .
```

For every spec with field constraints, append `-fields` to the spec's IRI
to name its field shape. The class shape or containing slot chooses the
value to check. The field shape supplies only the checks, applied with
`sh:node`. See
[Nested Object Slots](ObjectSlots.md#nested-object-slots) for the RDF
instance mapping and the slot shape.

### Type Matching and Field Validation

**Type matching (Xeto fitting)** checks whether a value matches a requested
type. For ordinary named types, matching is **nominal**: it uses the record's
declared type and its type relationships. Sugar also supports **structural
matching**, using the anchor type and the sugar's required tags and invariants.

**Field validation** checks a spec's slot requirements against a record's
fields, including required fields and value types. Neither nominal matching nor
structural sugar matching establishes that a record passes field validation:

- *Nominal matching* can accept a record's declared type even when the record
  lacks a required field. *Field validation* reports the missing field.
- For required markers, *structural sugar matching* requires the tag to be present
  but accepts any value type. *Field validation* requires a Marker value.
  For example, in the [nested sugar example](ObjectSlots.md#nested-sugar-types),
  a Target record with `hot: "oops"` structurally matches Hot because the `hot`
  tag exists, but fails field validation because the value is a string.

See also [reference constraints](References.md#sugar-reference-targets) for how
type matching applies to a Ref target.


### Abstract Specs

Translate abstract specs using the same class and field-constraint mappings as
other specs. The `abstract` marker does not add a SHACL constraint prohibiting
records from declaring that type directly. For example:

```xeto
Abstract: Dict <abstract> { label: Str }
```
translates to:
```turtle
ex:Abstract-fields a sh:NodeShape ;
  sh:property [
    sh:path ex:Abstract.label ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1
  ] .
```

### Constraints on the `spec` Field

In Trio, the `spec` field identifies the record's declared Xeto type.
Constraints on this field use the shared RDF property `sys:spec`, including
inherited constraints and returned-record validation. For example:

```xeto
Record: Dict {
  spec: Ref
}
```

The slot constraint translates to:

```turtle
ex:Record-fields a sh:NodeShape ;
  sh:property [
    sh:path sys:spec ;
    sh:nodeKind sh:IRI ;
    sh:class rdfs:Class ;
    sh:minCount 1 ;
    sh:maxCount 1
  ] .
```

This shape requires exactly one `sys:spec` value identifying an exported schema
class. It does not require that value to identify Record itself: a subtype
instance can refer to its own declared type. See [The `spec` Field](#the-spec-field)
for the instance mapping.

## Instance Data

A record available for lookup uses its Xeto id as its subject IRI. A library
instance uses its library's versioned namespace. An unqualified runtime id
uses the [project's base URI](Foundations.md#project-record-ids). The same mapping applies when that
id appears in a Ref or as a nested dictionary's `xetoRdf:sourceId`.

When an ordinary slot override supplies several property IRIs for one stored
field, each property points to the same RDF value. For a nested Dict or
quantity value this means sharing the blank node. For a list it means sharing
the [collection head](Lists.md#instance-data).

Each lookup record carries the RDF class `xetoRdf:Record`
and its effective spec. Ordinary Refs and Query execution use `xetoRdf:Record`
to recognize records available for lookup.

```xeto
@site1: Site {
  site
}

@equip1: Equip {
  dis: "AHU-1"
  siteRef: @site1
}
```

```turtle
ex:site1 a xetoRdf:Record, ex:Site ;
  sys:spec ex:Site ;
  xetoRdf:hasMarker ex:Site.site .

ex:equip1 a xetoRdf:Record, ex:Equip ;
  sys:spec ex:Equip ;
  ex:Equip.dis "AHU-1"^^xsd:string ;
  ex:Equip.siteRef ex:site1 .
```

A required `id: Ref`, including `id: Ref<of:Person>`, accepts an IRI subject
or a nested blank node with exactly one IRI-valued `xetoRdf:sourceId`. An
optional id permits the blank node to omit that property. See
[nested object slots](ObjectSlots.md#instance-data).

### The `spec` Field

Xeto instance syntax puts the declared type after the colon. The declaration
`@equip1: Equip` in the instance example identifies `Equip` as the instance's
type and implies a `spec` field referring to that definition. If `Equip` belongs
to library `example`, the corresponding `id` and `spec` fields in Trio are:

```trio
id: @example::equip1
spec: @example::Equip
```

The field identifies this record's declared type. Other Ref fields can also
refer to schema definitions; the special meaning of `spec` is that it names
the record's own type.

A Query can follow this Ref with `via:"spec"`. To preserve that traversal,
RDF needs a property containing the exact Ref in the field. `rdf:type` cannot
serve that purpose: it records type membership, which may include supertypes
or other type assertions. Following those memberships would introduce
traversal destinations that are not in the `spec` field.

The field therefore uses the shared RDF property `sys:spec`, separately from
`rdf:type`. The corresponding type and reference statements are:

```turtle
ex:equip1 a xetoRdf:Record, ex:Equip ;
  sys:spec ex:Equip .
```

An instance declared as a subtype identifies that subtype in its `spec` field,
not its parent type. It can therefore satisfy an inherited `spec: Ref`
requirement without referring to the parent: the requirement asks for a type
reference, not a reference to one particular type. Additional type memberships
must not create additional `sys:spec` values.

Following `sys:spec` looks for a runtime record with the referenced ID; the
schema definition alone is not a Query result. See
[Spec-Reference Traversal](QueryTraversal.md#spec-reference-traversal) for the selection rules
and examples.

Trio can also represent records without a `spec` field. If a record has no
stored `spec` field, the exporter must not add `sys:spec`, even when it assigns
the record an effective RDF type. Whether the field is required depends on
the applicable validation constraints. The same field mapping applies to
nested Dicts.

### Effective Instance Values

Before exporting an instance, the exporter looks up the complete definition of
its spec in the active namespace. That definition includes inherited and global
slots, defaults supplied by Xeto, implied markers, choices, and slots added by
mixins. Using it ensures that RDF export interprets each entry the same way as
Xeto validation.

During RDF export, each compiled Xeto instance is resolved against its effective
spec. Required default values and implied markers from that spec are included
in the instance's exported RDF representation.

The resolved instance values are the source of truth:

- If resolving the instance supplies a value, that value appears in RDF.
- If a value is still absent after resolution, the exporter does not write an
  RDF statement for it.

```xeto
Target : Dict { target }

Defaults : Dict {
  implied
  label: Str "ready"
  targetRef: Ref?<of:Target>
}

@defaults1: Defaults {}
```

Although `defaults1` is empty in the source, its resolved values include the
required `label` default and the `implied` marker. Both therefore appear in RDF.
`targetRef` is optional and has no value, so the RDF contains no `targetRef`
statement:

```turtle
ex:defaults1 a xetoRdf:Record, ex:Defaults ;
  ex:Defaults.label "ready"^^xsd:string ;
  xetoRdf:hasMarker ex:Defaults.implied .
```

This rule is the same for strings and other scalars, enums, units, references,
nested dictionaries, lists, and Choices. It also applies to defaults inherited
from a base spec, declared by a global slot, or contributed by a mixin. The
exporter writes such a default only when it is present in the resolved instance
values.

### Undeclared Fields

The following `Person` spec declares only `dis`:

```xeto
Person : Dict {
  dis: Str
}
```

In this instance, `dis` is a declared slot. `nickname` is an undeclared
string-valued entry, and `imported` is an undeclared marker entry:

```xeto
@person1: Person {
  dis: "Ann"
  nickname: "Annie"
  imported
}
```

```turtle
ex:person1 a xetoRdf:Record, ex:Person ;
  ex:Person.dis "Ann"^^xsd:string ;
  ex:Person.nickname "Annie"^^xsd:string ;
  xetoRdf:hasMarker ex:Person.imported .
```

An undeclared value-bearing entry uses a property IRI formed from the
instance's concrete spec and entry name. An undeclared marker uses
`xetoRdf:hasMarker` and the corresponding marker IRI. These extra entries do not
cause the open SHACL shape to reject an otherwise valid instance.
