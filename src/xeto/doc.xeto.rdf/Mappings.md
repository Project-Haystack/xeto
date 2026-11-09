# Mapping Rules

This chapter describes the shared mappings for specs, slots, and instances.
The [mapping chapters](doc.xeto.rdf::index) group each construct's RDF vocabulary, SHACL
validation, and instance-data mappings. A subsection is omitted when the
construct has no separate mapping for that profile.

## Specs, Slots, and Instances

### Vocabulary

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

A **value-bearing slot** is a slot whose instance value is represented as the
object of an RDF statement, such as a string, number, reference, nested value,
choice, or list. This distinguishes it from a marker, which uses
`sys:hasMarker`, and a computed `Query`, which may use a property path.

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
  sys:hasMarker ex:Site.site .

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
[SHACL Ref mapping](References.md#ref-and-multiref-slots) states that `ex:Equip.siteRef`
must point to an `ex:Site`, while the class declarations identify `Equip` and
`Site` themselves.

### SHACL Validation

A spec with instance constraints is a `sh:NodeShape` targeting the same IRI as
its RDF class.

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip .
```

Vocabulary relationships determine which shapes apply. For example, a subtype
inherits its supertypes' constraints, an intersection is a subtype of every
member, and mixin properties are attached directly to the target shape. The corresponding RDF statements are shown under
[Intersections and Unions](TypeComposition.md#intersections-and-unions) and [Mixins](TypeComposition.md#mixins).

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
[choices](MarkersAndChoices.md#choices), and [lists](Lists.md#lists).

#### Abstract Specs

Translate abstract specs using the same class and field-constraint mappings as
other specs. The `abstract` marker does not add a SHACL constraint prohibiting
records from declaring that type directly. For example:

```xeto
Abstract: Dict <abstract> { label: Str }
```
translates to:
```turtle
ex:Abstract a sh:NodeShape ;
  sh:targetClass ex:Abstract ;
  sh:property [
    sh:path ex:Abstract.label ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1
  ] .
```

#### Constraints on the `spec` Field

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
ex:Record a sh:NodeShape ;
  sh:targetClass ex:Record ;
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

### Instance Data

A Xeto instance id becomes the subject IRI. A local id uses the instance
library's versioned namespace; a qualified id uses the named library's
versioned namespace. The instance is both a `sys:Entity` and an instance of its
effective spec.

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
ex:site1 a sys:Entity, ex:Site ;
  sys:spec ex:Site ;
  sys:hasRefField sys:spec ;
  sys:hasMarker ex:Site.site .

ex:equip1 a sys:Entity, ex:Equip ;
  sys:spec ex:Equip ;
  sys:hasRefField sys:spec ;
  ex:Equip.dis "AHU-1"^^xsd:string ;
  ex:Equip.siteRef ex:site1 .
```

The subject IRI carries the Xeto id. No additional id predicate is required by
this profile.

#### The `spec` Field

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
ex:equip1 a sys:Entity, ex:Equip ;
  sys:spec ex:Equip ;
  sys:hasRefField sys:spec .
```

An instance declared as a subtype identifies that subtype in its `spec` field,
not its parent type. It can therefore satisfy an inherited `spec: Ref`
requirement without referring to the parent: the requirement asks for a type
reference, not a reference to one particular type. Additional type memberships
must not create additional `sys:spec` values.

Following `sys:spec` looks for a runtime record with the referenced ID; the
schema definition alone is not a Query result. See
[Spec-Reference Traversal](Queries.md#spec-reference-traversal) for the selection rules
and examples.

Trio can also represent records without a `spec` field. If a record has no
stored `spec` field, the exporter must not add `sys:spec`, even when it assigns
the record an effective RDF type. Whether the field is required depends on
the applicable validation constraints. The same field mapping applies to
nested Dicts.

#### Effective Instance Values

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
ex:defaults1 a sys:Entity, ex:Defaults ;
  ex:Defaults.label "ready"^^xsd:string ;
  sys:hasMarker ex:Defaults.implied .
```

This rule is the same for strings and other scalars, enums, units, references,
nested dictionaries, lists, and Choices. It also applies to defaults inherited
from a base spec, declared by a global slot, or contributed by a mixin. The
exporter writes such a default only when it is present in the resolved instance
values.

#### Undeclared Fields

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
ex:person1 a sys:Entity, ex:Person ;
  ex:Person.dis "Ann"^^xsd:string ;
  ex:Person.nickname "Annie"^^xsd:string ;
  sys:hasMarker ex:Person.imported .
```

An undeclared value-bearing entry uses a property IRI formed from the
instance's concrete spec and entry name. An undeclared marker uses
`sys:hasMarker` and the corresponding marker IRI. These extra entries do not
cause the open SHACL shape to reject an otherwise valid instance.

