# Overview

Xeto defines a standard mapping of specs and instance data to RDF 1.1. The
mapping uses RDFS for vocabulary relationships, SHACL Core for validation, and
QUDT identifiers for units and quantity kinds.

This chapter defines three profiles. A profile is a group of related mapping
rules that an exporter can support:

- the **vocabulary profile** maps Xeto specs and metadata to RDF resources;
- the **SHACL profile** maps Xeto validation semantics to SHACL shapes; and
- the **instance profile** maps Xeto instances and values to RDF data.

An exporter can support one or more profiles. A single Turtle file may contain
statements from all three profiles, or an exporter may write each profile to a
separate file. The profiles do not need to be separated into named graphs
inside an RDF dataset.

**Profile applicability.** Each construct later in this chapter includes only
the profile subsections that apply to it. If a construct has no `Vocabulary`,
`SHACL Validation`, or `Instance Data` subsection, that profile adds no separate
RDF statements for the construct. The omission is intentional; it does not mean
that the mapping is unfinished. Mappings that are not yet supported are listed
explicitly under [Deferred Mappings](#deferred-mappings).

# Conformance

The Xeto project maintains an implementation of this mapping for normal RDF
export. This chapter calls it the **reference exporter**. It supports all three
profiles, but this specification defines conformance; the reference exporter's
exact Turtle output does not.

An RDF export conforms to:

- the **vocabulary profile** when it contains every required vocabulary
  statement for each spec included in the export;
- the **SHACL profile** when it contains every required shape and constraint
  for each effective spec included in the export; and
- the **instance profile** when it contains every required data statement for
  each instance included in the export.

Although this specification aims to cover all Xeto constructs, some do not yet
have a mapping; see [Deferred Mappings](#deferred-mappings). Conformance is
fail-closed, an exporter must reject an unsupported construct rather than
silently omit it or emit weaker RDF or SHACL.

Another exporter may support any combination of these profiles and may include
additional descriptive RDF statements. Those additions must not conflict with
this mapping or change which instance data passes validation. For behavior
covered by this specification, output of additional SHACL constraints that reject
data accepted by Xeto, or omission of constraints that accept data rejected
by Xeto, do not conform to this specification.

The same RDF graph can be written as Turtle in several different ways. For
example, `ph:Equip` and `haystack:Equip` identify the same resource when both
prefixes expand to the same full IRI. Triple order, whitespace, and the local
labels assigned to blank nodes also do not change the graph's meaning. For this
reason, exporters are compared using the parsed RDF graph and SHACL validation
results rather than their Turtle text. Likewise, RDF-consuming applications
should therefore parse the RDF graph instead of comparing Turtle files line
by line.

This chapter defines what RDF an exporter produces, but it does not prescribe
how a user asks for that output. One tool might write vocabulary, SHACL shapes,
and instance data into one Turtle file. Another might provide separate commands
or options for producing three files. Both approaches conform when each
requested profile contains the statements defined here. No particular command
name, option name, file layout, or named-graph layout is required.

# Non-Goals

This mapping does not require:

- OWL inference;
- stored inverse relationship triples;
- byte-for-byte Turtle compatibility between exporters;
- copying complete external ontologies into every export; or
- a production dependency between Haxall and any independent conformance tool.

SPARQL query generation and optional relationship materialization are separate
concerns from RDF vocabulary, SHACL validation, and instance projection.

# Mapping Context

## Namespace

Xeto resolves names within a namespace: the set of libraries and exact library
versions currently in use. RDF export uses that same namespace so a name refers
to the same definition in Xeto and RDF.

A qualified name identifies its library directly. For example,
`base.people::Person` refers to `Person` from `base.people`, even when other
libraries are present. A simple name such as `Person` has no library part. Xeto
searches the namespace and uses the name only when exactly one library defines
it. No match is an unresolved-name error; more than one match is an
ambiguous-name error that must be fixed by using a qualified name.

For example, suppose the namespace contains `base.people` and `app.people`.
`base.people` defines `Person`; `app.people` defines `Org` and contributes a
mixin to `base.people::Person`, but does not define another `Person`. The simple
name `Person` therefore has one match and resolves to `base.people::Person`. If
`app.people` also declared its own `Person`, the simple name would be ambiguous;
neither library would automatically take priority.

## Complete Spec Definition

An exporter does not translate each Xeto source file in isolation. It exports
the complete definition of each spec as it exists in the current namespace.
This complete definition includes inherited slots, global-slot rules, choice
members, and mixins. This chapter calls that complete runtime definition the
**effective spec**.

For example, `base.people` might define:

```xeto
Person : Dict {
  dis: Str
}
```

Another library, `app.people`, might extend it:

```xeto
Org : Dict

+Person {
  orgRef: Ref<of:Org>
}
```

In a namespace containing only `base.people`, the RDF shape for `Person`
contains the `dis` constraint. In a namespace that also contains `app.people`,
the same `Person` shape contains both `dis` and `orgRef` constraints. The mixin
does not become a separate RDF class; it changes the exported shape of
`base.people::Person` in that namespace.

The [Mixins](#mixins) section describes how the contributed `orgRef` slot keeps
its `app.people` identity in RDF without creating a separate `app.people::Person`
class.

## Concrete Library Versions

A library declares its dependencies as library names plus acceptable version
ranges. When that library is compiled, its namespace contains concrete library
versions that satisfy those ranges. A project namespace is assembled by the
application from the libraries enabled for that project; Xeto does not prescribe
the application's installation or version-selection policy. By the time RDF is
exported, the namespace already contains exactly one selected version of each
library.

Every RDF IRI that comes from a Xeto library uses the version of the library
that defines the name. For example, `app.people` may accept any `1.0.x` version
of `base.people`, while the namespace selects version `1.0.3`. Names defined by
`base.people` are then written as follows:

```text
base.people::Person       -> http://xeto.dev/rdf/base.people-1.0.3#Person
base.people::Person.email -> http://xeto.dev/rdf/base.people-1.0.3#Person.email
base.people::person1      -> http://xeto.dev/rdf/base.people-1.0.3#person1
```

The dependency range `1.0.x` lists acceptable versions; it is not written into
an RDF IRI. The selected version `1.0.3` is used for classes, slots, Query
paths, Choice members, instance ids, instance classes, and references between
instances.

If the namespace instead selects `base.people` version `1.0.7`, the `Person`
IRI is `http://xeto.dev/rdf/base.people-1.0.7#Person`. The dependency may still
allow `1.0.x`; only the version selected by the namespace changes the RDF IRI.

Export requires exactly one version for every referenced library. For example:

- If `app.people` refers to `base.people::Person`, but `base.people` is missing
  from the namespace, export fails with `Concrete Xeto library version
  unavailable for base.people`.
- If the export input contains both `base.people` version `1.0.3` and version
  `1.0.7`, export fails with `Ambiguous Xeto library version for base.people:
  1.0.3 and 1.0.7`.

# RDF Graphs

Each Turtle output document represents one RDF graph. An exporter may write
selected items to separate files or place their RDF statements together in one
file. Combining them does not change their IRIs or mapping rules.

The same IRI identifies both a Xeto spec's RDF class and its SHACL shape. For
example:

```turtle
base:Person a rdfs:Class, sh:NodeShape ;
  sh:targetClass base:Person .
```

This says that `base:Person` is a class and that the constraints attached to
it validate instances of that class. The exporter does not create a second IRI
such as `base:PersonShape`.

SHACL calls this an **open shape**: it validates the predicates named by the
shape and ignores other predicates on the same instance. This matches Xeto
validation, which accepts entries in an instance dictionary that are not
declared as slots while continuing to validate every recognized slot. This is
general dictionary behavior, not a special case for mixins or marker tags.

For example, a `Person` shape can require `base:Person.dis` and still accept an
undeclared `nickname` entry. It continues to reject the instance if the required
`dis` entry is missing or has the wrong type. The exporter therefore does not
add `sh:closed true` unless a later mapping rule explicitly requires it.

# Namespaces and IRIs

Every Xeto library version has this RDF namespace IRI:

```text
http://xeto.dev/rdf/{lib-name}-{version}#
```

This specification assigns `http://xeto.dev/rdf/` as the base for Xeto RDF
namespaces. It is a Xeto convention, not a requirement imposed by RDF itself.

For example, version `5.0.0` of `ph.points` uses:

```text
http://xeto.dev/rdf/ph.points-5.0.0#
```

The namespace IRI uses the concrete version selected by the Xeto namespace.
The full library name is copied as written, including dots. For example,
`lib.name` appears as `lib.name-1.2.3` in its versioned namespace.

| Xeto name or namespace | Full RDF IRI |
| --- | --- |
| namespace for library `lib.name`, version `1.2.3` | `http://xeto.dev/rdf/lib.name-1.2.3#` |
| spec `lib.name::Equip` | `http://xeto.dev/rdf/lib.name-1.2.3#Equip` |
| slot `lib.name::Equip.siteRef` | `http://xeto.dev/rdf/lib.name-1.2.3#Equip.siteRef` |
| marker slot `lib.name::Equip.equip` | `http://xeto.dev/rdf/lib.name-1.2.3#Equip.equip` |
| instance `@lib.name::equip1` | `http://xeto.dev/rdf/lib.name-1.2.3#equip1` |

Turtle prefixes provide a shorter way to write full IRIs. For example:

```turtle
@prefix lib: <http://xeto.dev/rdf/lib.name-1.2.3#> .

lib:equip1 a lib:Equip .
```

That statement means exactly the same as:

```turtle
<http://xeto.dev/rdf/lib.name-1.2.3#equip1>
  a <http://xeto.dev/rdf/lib.name-1.2.3#Equip> .
```

The prefix name `lib:` is only an abbreviation. Calling it `example:` instead
would not change the full IRIs or the meaning of the statement.

When Xeto code uses a fully qualified name such as `lib.name::Equip`, the
library is already explicit. When it uses the simple name `Equip`, Xeto first
finds the matching spec in the current namespace. In both cases, the exporter
ends with the same fully qualified Xeto name and builds the same RDF IRI.

The following prefixes are used by the Turtle examples and qualified names in
this chapter:

```turtle
@prefix rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix owl:  <http://www.w3.org/2002/07/owl#> .
@prefix sh:   <http://www.w3.org/ns/shacl#> .
@prefix xsd:  <http://www.w3.org/2001/XMLSchema#> .
@prefix qudt: <http://qudt.org/schema/qudt/> .
@prefix unit: <http://qudt.org/vocab/unit/> .
@prefix currency: <http://qudt.org/vocab/currency/> .
@prefix quantitykind: <http://qudt.org/vocab/quantitykind/> .
@prefix skos: <http://www.w3.org/2004/02/skos/core#> .
@prefix sys:  <http://xeto.dev/rdf/sys-5.0.0#> .
@prefix ex:   <http://xeto.dev/rdf/example-1.0.0#> .
```

The `sys:` and `ex:` versions above are examples. An actual export uses the
concrete versions selected by its Xeto namespace. The `owl:` prefix is used only
for the optional ontology-packaging terms described below; the mapping itself
does not require OWL inference.

## Turtle-safe names and values

Turtle uses characters such as quotes and backslashes as syntax. When those
characters occur in Xeto text, enum keys, patterns, or documentation, the
exporter escapes them without changing the value represented by the RDF graph.

```xeto
State : Enum {
  unusual <key:"line\n\"quoted\"\\slash Ω">
}

Record : Dict <doc:"""A "quoted" state
on the next line"""> {
  state: State
}
```

```turtle
ex:Record a sh:NodeShape ;
  rdfs:comment "A \"quoted\" state\non the next line"@en ;
  sh:property [
    sh:path ex:Record.state ;
    sh:in ("line\n\"quoted\"\\slash Ω"^^xsd:string) ;
  ] .
```

The Xeto documentation contains literal quote characters and spans two source
lines. Turtle writes those characters as `\"` and `\n` inside one short string
literal. Likewise, the enum key's `\n`, `\"`, and `\\` sequences represent its
newline, quote, and backslash characters. An RDF reader recovers the original
values. Unicode characters such as `Ω` retain their value. The same escaping
rule applies to language-tagged labels and documentation.

Haystack stores a `Number` as a binary floating-point value with an optional
unit. Xeto `Float` is the unitless subtype of `Number`; it does not introduce a
different numeric representation. Both types therefore map to `xsd:double`.
`Int` remains `xsd:integer` because its Xeto type permits only whole numbers.

The exporter first parses a finite Number or Float using this binary value
model, then writes a stable double lexical form. This matters when the source
contains more digits than the binary value can retain:

```xeto
Reading : Dict {
  value: Number
  ratio: Float
  count: Int
}

@reading1: Reading {
  value: 0.10000000000000001
  ratio: 1e-7
  count: 5
}
```

```turtle
ex:reading1 a sys:Entity, ex:Reading ;
  ex:Reading.value "0.1"^^xsd:double ;
  ex:Reading.ratio "1.0E-7"^^xsd:double ;
  ex:Reading.count "5"^^xsd:integer .
```

The source spelling `0.10000000000000001` resolves to the same binary value as
`0.1`, so the shorter round-trippable spelling is exported. Exponent notation
may be used for very small or large values.

`NaN`, `INF`, and `-INF` map to the corresponding `xsd:double` values. `NaN`
cannot be a `minVal` or `maxVal` because it has no numeric order. A malformed
number is an export error; the exporter does not copy a valid-looking numeric
prefix from an invalid value.

Most Xeto names can be written using a Turtle prefix. If a legal Xeto instance
id is not safe in that short form, the exporter writes the same name as a full
versioned IRI instead. For example:

```xeto
@name~one: Reading {}
```

```turtle
<http://xeto.dev/rdf/example-1.0.0#name~one> a ex:Reading .
```

Using a full IRI changes only the Turtle spelling; the instance keeps the same
RDF identity.

# Metadata and External Vocabularies

The RDF mapping does not require OWL reasoning. An exporter may describe a
library resource as an `owl:Ontology` and may use `owl:imports` as packaging
metadata, but these statements are not required for vocabulary, SHACL, or
instance-profile conformance.

Each value-bearing Xeto slot becomes an RDF property that connects an instance
to a value. For example, `siteRef` connects an equipment instance to its site:

```turtle
lib:equip1 lib:Equip.siteRef lib:site1 .
```

The vocabulary export also describes that property so RDF tools can recognize
and display it:

```turtle
lib:Equip.siteRef a rdf:Property ;
  rdfs:label "siteRef"@en .
```

The first statement contains instance data. The second identifies
`lib:Equip.siteRef` as an RDF property and gives it the English label
`siteRef`; the `@en` suffix marks the label as English. Every exported slot
property includes this description. Its SHACL constraints are defined
separately.

Xeto names and documentation map like this:

```xeto
Person : Dict <doc:"A person">
```

```turtle
lib:Person rdfs:label "Person"@en ;
  rdfs:comment "A person"@en .
```

In an RDF statement, the predicate names the relationship between the subject
and object. In this example, `rdfs:label` and `rdfs:comment` are the predicates.
They relate `lib:Person` to its display label and documentation text.

## Metadata Without an RDF Mapping

Xeto libraries can add their own metadata to specs and slots. For example, an
application can define an `icon` tag and use it to select an icon in its user
interface:

```xeto
+Spec {
  icon: Str?
}

Person : Dict <icon:"user"> {
  dis: Str <icon:"label">
}
```

This specification does not assign an RDF meaning to `icon`, so the RDF is the
same as if the `icon` metadata were not present:

```turtle
lib:Person a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "Person"@en .

lib:Person.dis a rdf:Property ;
  rdfs:label "dis"@en .
```

The `+Spec` declaration does not create an RDF property or a SHACL shape for
`icon`, and the two uses of `icon` do not create RDF statements.

An exporter omits metadata for which this specification defines no mapping. It
still exports the containing spec or slot. This rule applies to
application-defined metadata and to standard Xeto metadata such as `summary` or
`readonly` when no mapping is defined here.

Metadata used by a mapping in this specification is handled strictly. For
example, `minVal`, `of`, `via`, `quantity`, and `unit` affect the generated
SHACL graph. An exporter reports an error when mapped metadata has an invalid
type or value; it does not ignore that error and emit a weaker constraint.

References to external vocabularies use their canonical published IRIs. Xeto
vocabulary, SHACL, and instance exports do not copy declarations or
classification facts owned by those vocabularies. A validation environment
loads the required external vocabulary graphs separately.

For example, a temperature unit shape may require a QUDT unit with the
temperature quantity kind:

```turtle
lib:Sensor a sh:NodeShape ;
  sh:property [
    sh:path lib:Sensor.unit ;
    sh:class qudt:Unit ;
    sh:node [
      sh:property [
        sh:path (
          qudt:hasQuantityKind
          [ sh:zeroOrMorePath skos:broader ]
        ) ;
        sh:hasValue quantitykind:Temperature ;
      ] ;
    ] ;
  ] .
```

During validation, the validator also needs the relevant facts from QUDT, such
as the unit's type and quantity-kind relationship:

```turtle
unit:DEG_F a qudt:Unit ;
  qudt:hasQuantityKind quantitykind:Temperature .
```

The path also accepts more specific QUDT quantity kinds. This rule and its
required QUDT data are explained under
[Units and Quantities](#quantity-kind-hierarchy).

The validator loads these facts from QUDT separately. It loads both the unit
graph and, when currency values are present, the currency graph. An instance
export may refer to `unit:DEG_F` or `currency:USD`, but it does not copy
QUDT-owned facts into the instance graph. The exact correspondence is defined
by the canonical machine-readable
[unit mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-units.props)
and
[quantity-kind mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-quantities.props)
packaged by Xeto's `sys.rdf` library.

# Mapping Rules

Each Xeto construct below groups its RDF vocabulary, SHACL validation, and
instance-data mappings. A subsection is omitted when the construct has no
separate mapping for that profile.

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
[SHACL Ref mapping](#ref-and-multiref-slots) states that `ex:Equip.siteRef`
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
[Intersections and Unions](#intersections-and-unions) and [Mixins](#mixins).

Ordinary Xeto slots use these cardinalities:

| Xeto slot | SHACL cardinality |
| --- | --- |
| required, single-valued | `sh:minCount 1`; `sh:maxCount 1` |
| `maybe`, single-valued | no minimum; `sh:maxCount 1` |
| required `MultiRef` | `sh:minCount 1`; no maximum |
| `maybe MultiRef` | no minimum or maximum |
| `Query` | no minimum or maximum |

The type and value constraints still apply when a `maybe` slot is present.
Marker, choice, and list cardinality have additional rules described in their
sections below.

#### Constraints on the `spec` Field

The type in a Xeto instance declaration, such as `@meter1: ElecMeter {}`,
implies a `spec` field referring to `ElecMeter`. When that name resolves to
`ph::ElecMeter`, a Trio representation expresses the field explicitly as
`spec: @ph::ElecMeter`. These are two ways to declare the same instance type;
see [Instance Specs](Instances.md#spec).

A type definition can declare requirements for the `spec` field using a slot
declaration, just as it does for other fields. For example:

```xeto
Record: Dict {
  spec: Ref
}
```

The [`spec` field](#the-spec-field) identifies the record's declared type and
is exported as `sys:spec`. SHACL therefore checks that property, not an
owner-qualified property such as `ex:Record.spec`:

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

The shape requires exactly one `sys:spec` value identifying an exported schema
class. It checks that the field contains a reference to a type definition; it
does not select which type the record declares.

Inherited requirements and reusable constraints for returned Query records
check the same `sys:spec` property. See [The `spec` Field](#the-spec-field) for
how subtype instances satisfy inherited requirements.

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
[Spec-Reference Traversal](#spec-reference-traversal) for the selection rules
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

## Standard Scalar Slots

A plain Xeto scalar maps to an RDF typed literal. A typed literal contains a
lexical form and a datatype. The quoted text in Turtle is the lexical form; it
does not mean that every scalar is an `xsd:string`.

The scalar's effective Xeto type determines the datatype. The exporter does not
infer the datatype from the value's appearance. For example:

```xeto
Reading : Dict {
  label: Str
  count: Int
  value: Number
  ratio: Float
}

@reading1: Reading {
  label: "5"
  count: 5
  value: 5
  ratio: 5e-1
}
```

```turtle
ex:reading1 a sys:Entity, ex:Reading ;
  ex:Reading.label "5"^^xsd:string ;
  ex:Reading.count "5"^^xsd:integer ;
  ex:Reading.value "5.0"^^xsd:double ;
  ex:Reading.ratio "0.5"^^xsd:double .
```

The same type mapping selects the SHACL datatype:

| Xeto type | RDF datatype |
| --- | --- |
| `Str` | `xsd:string` |
| `Number` | `xsd:double` |
| `Float` | `xsd:double` |
| `Int` | `xsd:integer` |
| `Bool` | `xsd:boolean` |
| `Date` | `xsd:date` |
| `Time` | `xsd:time` |
| `DateTime` | `xsd:dateTime` |
| `Uri` | `xsd:anyURI` |
| `TimeZone` | `xsd:string` |

### SHACL Validation

A unit bearing `Number` is an exception, it maps to a `qudt:QuantityValue` node
rather than a plain literal, as described under
[Units and Quantities](#units-and-quantities).

`Float` is always unitless in Xeto and therefore never uses the QUDT quantity
mapping. Number and Float share the same finite and special double values:
`"NaN"^^xsd:double`, `"INF"^^xsd:double`, and `"-INF"^^xsd:double`. Negative
zero retains its sign. `NaN` cannot be used for `minVal` or `maxVal` because it
has no numeric order.

Numeric `minVal` and `maxVal` map to inclusive bounds.

```xeto
Sensor : Dict {
  sampleInterval: Int <minVal:1, maxVal:60>
}
```

```turtle
ex:Sensor a sh:NodeShape ;
  sh:targetClass ex:Sensor ;
  sh:property [
    sh:path ex:Sensor.sampleInterval ;
    sh:datatype xsd:integer ;
    sh:minInclusive 1 ;
    sh:maxInclusive 60 ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

String `minSize`, `maxSize`, and `nonEmpty` map to `sh:minLength`,
`sh:maxLength`, and `sh:pattern`. `nonEmpty` uses the pattern `\\S`, which
requires at least one non-whitespace character. Explicit `pattern` metadata
also maps to `sh:pattern`.

```xeto
Asset : Dict {
  dis: Str <nonEmpty, minSize:2, maxSize:40>
}
```

```turtle
ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.dis ;
    sh:datatype xsd:string ;
    sh:minLength 2 ;
    sh:maxLength 40 ;
    sh:pattern "\\S" ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

An invariant slot value maps to `sh:hasValue` with the scalar's RDF datatype.
An ordinary default value does not constrain instance data.

```xeto
Thing : Dict {
  fixed: Str <invariant> "ok"
  label: Str "fallback"
}
```

```turtle
ex:Thing a sh:NodeShape ;
  sh:targetClass ex:Thing ;
  sh:property [
    sh:path ex:Thing.fixed ;
    sh:datatype xsd:string ;
    sh:hasValue "ok"^^xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path ex:Thing.label ;
    sh:datatype xsd:string ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

### Instance Data

Scalar values use the datatypes from the scalar mapping table. For example:

```xeto
Sensor : Dict {
  dis: Str
  priority: Int
  enabled: Bool
  commissioned: Date
}

@sensor1: Sensor {
  dis: "Zone Temp"
  priority: 5
  enabled: "true"
  commissioned: 2026-07-19
}
```

```turtle
ex:sensor1 a sys:Entity, ex:Sensor ;
  ex:Sensor.dis "Zone Temp"^^xsd:string ;
  ex:Sensor.priority "5"^^xsd:integer ;
  ex:Sensor.enabled "true"^^xsd:boolean ;
  ex:Sensor.commissioned "2026-07-19"^^xsd:date .
```

## Custom Scalar Datatypes

A custom Xeto scalar is a named type derived from `Scalar`, directly or through
another scalar type, with its own value constraints or encoding.

A concrete Xeto scalar without a specialized RDF mapping maps to a custom RDF
datatype. Its datatype IRI is the scalar spec's IRI in its versioned library
namespace. For example, `Code` in the example library maps to `ex:Code`.

Each value maps to an RDF literal whose datatype is that IRI and whose lexical
form is Xeto's string encoding of the value: `"AB"^^ex:Code`. This encoding
represents the scalar value; it does not imply that the value is a string.
The encoded form may differ from the original Xeto source spelling.

The mapping preserves type identity and encoded content. SHACL enforces
explicit constraints such as patterns, but does not reproduce native parsing,
comparison, execution, or resource lookup. These rules apply equally to
user-defined scalars and the built-in scalars listed below.

Types with an existing special mapping keep that mapping, including
refinements of [standard scalars](#standard-scalar-slots), enums, markers,
references, and units. The built-in scalars covered by the custom datatype
rule are listed [below](#built-in-and-library-scalars).

### Vocabulary

The vocabulary declares the scalar spec's IRI as an `rdfs:Datatype`:

```xeto
Code : Scalar <pattern:"[A-Z]{2}">

Asset : Dict {
  code: Code
}
```

```turtle
ex:Code a rdfs:Datatype ;
  rdfs:label "Code"@en .
```

The declaration names the datatype. The following shape enforces its pattern
when a value is used in `Asset.code`.

### SHACL Validation

A custom scalar slot checks the value's datatype with `sh:datatype` and its
declared pattern with `sh:pattern`. Required and optional slots follow the
ordinary cardinality rules.

```turtle
ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.code ;
    sh:datatype ex:Code ;
    sh:pattern "[A-Z]{2}" ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

A custom scalar subtype maps to its own RDF datatype. Its shape preserves
both its declared patterns and those inherited from its base type:

```xeto
UpperCode : Scalar <pattern:"[A-Z]+">
FourCharacterCode : UpperCode <pattern:"[A-Z0-9]{4}">

Asset : Dict {
  code: FourCharacterCode
}
```

```turtle
ex:UpperCode a rdfs:Datatype ;
  rdfs:label "UpperCode"@en .

ex:FourCharacterCode a rdfs:Datatype ;
  rdfs:label "FourCharacterCode"@en ;
  rdfs:subClassOf ex:UpperCode .

ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.code ;
    sh:datatype ex:FourCharacterCode ;
    sh:and (
      [ sh:pattern "[A-Z]+" ]
      [ sh:pattern "[A-Z0-9]{4}" ]
    ) ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

The slot's pattern constraints come from its declared scalar type, that type's
base types, and any pattern declared directly on the slot. Every applicable
pattern must be satisfied. Multiple distinct patterns map to separate shapes
within `sh:and`. SHACL permits only one `sh:pattern` value per shape under its
[multiple-parameter rule](https://www.w3.org/TR/shacl/#multiple-parameters).

An invariant custom scalar value maps to `sh:hasValue` with the slot's
datatype:

```xeto
Code : Scalar <pattern:"[A-Z]{2}">

FixedAsset : Dict {
  code: Code <invariant> "AB"
}
```

```turtle
ex:Code a rdfs:Datatype ;
  rdfs:label "Code"@en .

ex:FixedAsset a sh:NodeShape ;
  sh:targetClass ex:FixedAsset ;
  sh:property [
    sh:path ex:FixedAsset.code ;
    sh:datatype ex:Code ;
    sh:pattern "[A-Z]{2}" ;
    sh:hasValue "AB"^^ex:Code ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

`"AB"^^ex:Code` and `"AB"^^xsd:string` have the same text but different
datatypes. Only the first satisfies this invariant. An ordinary default does
not produce an `sh:hasValue` constraint.

### Instance Data

Using the `Code` and `Asset` declarations above:

```xeto
@asset1: Asset {
  code: "AB"
}
```

```turtle
ex:asset1 a sys:Entity, ex:Asset ;
  ex:Asset.code "AB"^^ex:Code .
```

For a slot declared with a custom scalar type, the slot's effective type
determines the literal's datatype. A slot declared as `UpperCode` therefore
uses `ex:UpperCode`, matching its `sh:datatype` constraint. The same rule
applies to supported list items with a declared scalar type and to scalar
slots in nested values.

An `Obj` slot does not declare a specific scalar type. The RDF datatype
therefore comes from the stored value's Xeto type. An explicitly typed `Code`
value retains that type; an ordinary string remains `xsd:string`:

```xeto
Envelope : Dict {
  value: Obj
}

@coded: Envelope {
  value: Code "AB"
}

@plain: Envelope {
  value: "AB"
}
```

```turtle
ex:coded a sys:Entity, ex:Envelope ;
  ex:Envelope.value "AB"^^ex:Code .

ex:plain a sys:Entity, ex:Envelope ;
  ex:Envelope.value "AB"^^xsd:string .
```

The export input must preserve a custom scalar value's type information.
If that information is missing, the exporter reports an error rather than
converting the custom value to an ordinary string.

### Built-in and Library Scalars

The same rule covers built-in and library scalars without another special
mapping. For example:

```turtle
sys:Version a rdfs:Datatype .
sys:Buf a rdfs:Datatype .

ex:record ex:version "1.2.3"^^sys:Version ;
  ex:payload "-_8"^^sys:Buf .
```

The text preserves what Xeto encoded. It does not make RDF tools understand
version ordering or binary data. `Duration` also uses this rule, even though
its Xeto base is `Number`.

| Other scalar | RDF representation | What RDF does not do |
| --- | --- | --- |
| `Version` | Xeto version text with datatype `sys:Version` | Compare version parts or check native parser limits |
| `Buf` | Xeto base64url text with datatype `sys:Buf` | Convert to `xsd:base64Binary` or decode the bytes |
| `Duration` | Xeto duration text with datatype `sys:Duration` | Perform time arithmetic or convert to a QUDT quantity |
| `Span` | Xeto span text with datatype `sys:Span` | Resolve relative dates or preserve context absent from the encoding |
| `Filter` | Xeto filter text with datatype `sys:Filter` | Parse, execute, or translate the filter to SPARQL |
| `LibDependVersions`, `LibFilePattern` | Selector text with the corresponding Xeto datatype | Resolve dependencies or inspect files |
| `None`, `NA` | The explicit value's Xeto encoding with its own datatype | Treat the value as omission, JSON null, or numeric NaN |
| `BuildVar` | A surviving placeholder's encoding with datatype `sys:BuildVar` | Perform build substitution |
| Other scalars, such as `pi::Icon`, `ph::Coord`, and user-defined `Code` | Xeto-encoded text with the scalar's versioned type IRI | Look up resources, split values into fields, or add type-specific parsing |

Values already replaced or removed by Xeto are not recreated during export.
A resolved build variable uses the mapping for its resulting value.

## Other Core Types

Most dictionary types need no special mapping. A type derived from `Dict` uses
the ordinary class, nested-object, and slot rules in this chapter. For example,
`Err` is a dictionary with `dis` and optional `errTrace` slots:

```xeto
Failure : Err

Envelope : Dict {
  failure: Failure?
}
```

```turtle
ex:Failure a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Err ;
  rdfs:label "Failure"@en .

ex:Envelope a sh:NodeShape ;
  sh:targetClass ex:Envelope ;
  sh:property [
    sh:path ex:Envelope.failure ;
    sh:node ex:Failure ;
    sh:maxCount 1 ;
  ] .
```

The following core types do not have mappings in this version:

| Xeto type | Reason |
| --- | --- |
| `Grid` | A grid mapping must define rows, columns, cells, metadata, and ordering. |
| `Collection` | The abstract collection type does not identify a concrete RDF value representation. |
| `Func` | Function parameters, return values, and execution are outside the current data profiles. |
| `Interface`, `Funcs`, and `File` | API contracts, callable operations, and file handles are outside the current data profiles. |

An exporter reports that the affected spec is unsupported when one of these
types is used as an RDF value type. It does not treat the value as a string,
dictionary, or unconstrained object. `Entity`, `Err`, and other ordinary
dictionary types continue to use the `Dict` mapping.

This check follows the complete inheritance chain. For example, a
`SpecialGrid` derived from `Grid` is still a grid value, so a slot whose type is
`SpecialGrid` fails with the unsupported Grid diagnostic. Giving an unsupported
core type an application-specific name does not turn it into a dictionary or
an unconstrained object.

## Unconstrained Object Slots

### SHACL Validation

An `Obj` slot constrains presence and single-value cardinality but deliberately
does not constrain the RDF term's datatype, class, or node kind.

```xeto
Envelope : Dict {
  value: Obj
}
```

```turtle
ex:Envelope a sh:NodeShape ;
  sh:targetClass ex:Envelope ;
  sh:property [
    sh:path ex:Envelope.value ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

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

## Enums

An enum's entry names define its allowed string values unless an entry supplies
an explicit `key`.

```xeto
Suit : Enum {
  clubs
  diamonds
}

Card : Dict {
  suit: Suit
}
```

### Vocabulary

An enum spec is exported as a class, and the slot using it is exported as an
RDF property. Ordinary enum values are strings rather than RDF individuals,
so the vocabulary graph does not create a resource for each entry.

Some built-in enums have their own RDF mappings instead of this default string
mapping. For example, `Unit` and
[UnitQuantity](#standalone-unitquantity-values) represent their values as
resources, as described in their respective sections.

```turtle
ex:Suit a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Enum ;
  rdfs:label "Suit"@en .

ex:Card.suit a rdf:Property ;
  rdfs:label "suit"@en .
```

### SHACL Validation

Ordinary enum slots are string-valued and use `sh:in` for the allowed values. For the
unkeyed `Suit` above, the entry names are used directly:

```turtle
ex:Card a sh:NodeShape ;
  sh:targetClass ex:Card ;
  sh:property [
    sh:path ex:Card.suit ;
    sh:datatype xsd:string ;
    sh:in ("clubs"^^xsd:string "diamonds"^^xsd:string) ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

If an enum entry declares `key` metadata, that key becomes the allowed value:

```xeto
Suit : Enum {
  clubs    <key:"Clubs">
  diamonds <key:"Diamonds">
}

Card : Dict {
  suit: Suit
}
```

The vocabulary declarations are unchanged. Only the `sh:in` values differ:

```turtle
ex:Card a sh:NodeShape ;
  sh:targetClass ex:Card ;
  sh:property [
    sh:path ex:Card.suit ;
    sh:datatype xsd:string ;
    sh:in ("Clubs"^^xsd:string "Diamonds"^^xsd:string) ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

The allowed values are therefore `"Clubs"` and `"Diamonds"`, not `"clubs"`
and `"diamonds"`.

Xeto enum definitions cannot be inherited. The Xeto compiler rejects a spec
that attempts to derive from an enum, so an enum slot's allowed values come
from the enum named by that slot. There is no additional enum-inheritance rule
in the RDF mapping.

### Instance Data

For the unkeyed enum, an instance uses the entry name as its string value:

```xeto
@card1: Card {
  suit: "clubs"
}
```

```turtle
ex:card1 a sys:Entity, ex:Card ;
  ex:Card.suit "clubs"^^xsd:string .
```

For the keyed definition, the instance and RDF literal use the explicit key:

```xeto
@card2: Card {
  suit: "Clubs"
}
```

```turtle
ex:card2 a sys:Entity, ex:Card ;
  ex:Card.suit "Clubs"^^xsd:string .
```

## Ref and MultiRef Slots

### SHACL Validation

A `Ref` value is an IRI. A typed ref requires the referenced resource to belong
to the target class. An untyped ref requires `sys:Entity`, which also prevents a
dangling IRI from satisfying the shape.

```xeto
Site : Dict

Equip : Dict {
  siteRef: Ref<of:Site>
  related: MultiRef?<of:Equip>
  auxRef: Ref?
}
```

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    sh:path ex:Equip.siteRef ;
    sh:nodeKind sh:IRI ;
    sh:class ex:Site ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path ex:Equip.related ;
    sh:nodeKind sh:IRI ;
    sh:class ex:Equip ;
  ] ;
  sh:property [
    sh:path ex:Equip.auxRef ;
    sh:nodeKind sh:IRI ;
    sh:class sys:Entity ;
    sh:maxCount 1 ;
  ] .
```

Every value of a `MultiRef` must satisfy the node-kind and class constraints.

### Instance Data

A `Ref` becomes one object IRI. A `MultiRef` becomes one triple per referenced
IRI using the same property.

```xeto
Site : Dict

Equip : Dict {
  siteRef: Ref<of:Site>
  related: MultiRef?<of:Equip>
}

@site1: Site {}
@equip2: Equip { siteRef: @site1 }
@equip3: Equip { siteRef: @site1 }

@equip1: Equip {
  siteRef: @site1
  related: MultiRef {
    @equip2
    @equip3
  }
}
```

```turtle
ex:site1 a sys:Entity, ex:Site .

ex:equip2 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 ;
  sys:hasRefField ex:Equip.siteRef .

ex:equip3 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 ;
  sys:hasRefField ex:Equip.siteRef .

ex:equip1 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 ;
  ex:Equip.related ex:equip2, ex:equip3 ;
  sys:hasRefField ex:Equip.siteRef, ex:Equip.related .
```

The single `siteRef` value produces one statement. The two `related` values
produce two statements with the same subject and property. Referenced Xeto
entities are exported with their own `sys:Entity` and concrete spec types so
typed and untyped reference shapes can validate them.

#### Reference-field metadata

RDF uses IRIs for Refs, Units, UnitQuantity values, and named nested Dicts.
A link alone does not show which Xeto value produced it. `sys:hasRefField`
identifies a Ref or MultiRef field on the record that stores it; the value of
`sys:hasRefField` is the field's property IRI.

| Stored Xeto value | Xeto Query behavior |
| --- | --- |
| Ref to an existing record | Follow it and consider that record for results. |
| Ref to a missing record | Return no result from that branch. |
| Unit, UnitQuantity, or named nested Dict | Fail if the Query tries to follow it as a Ref. |

Without this metadata, SPARQL could return an empty result for the last case
where Xeto reports an error. Query input validation uses `sys:hasRefField` to
preserve that distinction.

A Ref contains the ID of another record. That referenced record need not be
present in the dataset. In library `example.people`, the metadata belongs to
the record holding the Ref (`ex:` denotes that library's RDF namespace):

The following RDF excerpts show instance data; schema triples are omitted.

```xeto
Person: Dict { address: Ref }
@person1: Person { address: Ref "example.people::missingAddress" }
```

```turtle
ex:person1 a sys:Entity, ex:Person ;
  ex:Person.address ex:missingAddress ;
  sys:hasRefField ex:Person.address .
# No record for ex:missingAddress is present in this graph.
```

The graph contains the IRI `ex:missingAddress`, but no record describing that
address, so this branch returns no result. `sys:hasRefField` describes the Ref
stored on person1, not whether the address record exists. Separate record
validation may reject the dangling Ref.

Before following a field with values, Query input validation requires its
`sys:hasRefField` statement. For example, following `address` on person1 above
passes this check; removing the statement makes it fail. This prevents a nested
Dict or a Unit IRI such as `unit:PA` from being mistaken for a Ref. An absent
field or an empty MultiRef has nothing to follow and needs no statement.
Consumers must preserve these statements when copying or transforming RDF.

Exporters use the stored value to decide whether to add `sys:hasRefField`,
including for `Obj` slots and undeclared fields. If a field is exported under
several property IRIs, such as inherited aliases, the exporter lists each
property IRI as a value of `sys:hasRefField` on the owning record.
For example, an override narrows an inherited Ref to an Address Ref. The
export retains both property IRIs and identifies both as reference fields:

```xeto
Address: Dict {}
Person: Dict { address: Ref }
Employee: Person { address: Ref<of:Address> }
@address1: Address {}
@employee1: Employee { address: @address1 }
```

```turtle
ex:address1 a sys:Entity, ex:Address .
ex:employee1 a sys:Entity, ex:Employee ;
  ex:Person.address ex:address1 ;
  ex:Employee.address ex:address1 ;
  sys:hasRefField ex:Person.address, ex:Employee.address .
```

For an explicitly stored empty MultiRef, the exporter lists the field's
property IRI under `sys:hasRefField`, but there are no referenced IDs to export.
An absent field produces neither reference values nor `sys:hasRefField` metadata:

```xeto
Group: Dict { members: MultiRef? }
@group1: Group { members: MultiRef {} }
```

```turtle
ex:group1 a sys:Entity, ex:Group ;
  sys:hasRefField ex:Group.members .
# No ex:Group.members values: the MultiRef is empty.
```

## Nested Object Slots

### SHACL Validation

A slot typed as another dictionary spec uses `sh:node` to apply that spec's
shape to the nested value.

```xeto
Address : Dict {
  city: Str
}

PostalAddress : Address

Person : Dict {
  address: Address
  postalAddress: PostalAddress?
}
```

```turtle
ex:Person a sh:NodeShape ;
  sh:targetClass ex:Person ;
  sh:property [
    sh:path ex:Person.address ;
    sh:node ex:Address ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path ex:Person.postalAddress ;
    sh:node ex:PostalAddress ;
    sh:maxCount 1 ;
  ] .
```

Dictionary inheritance may pass through more than one spec. The exporter
follows the complete inheritance chain, but `sh:node` names the slot's declared
type; in this example, `PostalAddress`. If that type is declared by a dependency,
the same rule applies using the dependency library's RDF namespace. A loaded
`addresses::PostalAddress`, for example, becomes
`sh:node addresses:PostalAddress`.

The nested node may itself contain scalar, ref, marker, unit, list, choice, or
further nested values supported by this specification.

### Instance Data

A nested dictionary without its own Xeto id becomes an RDF blank node. It is
typed with its nested spec so both class-based and `sh:node` constraints can
recognize it.

```xeto
@person1: Person {
  address: Address {
    city: "Richmond"
  }
  postalAddress: PostalAddress {
    city: "Richmond"
  }
}
```

```turtle
ex:person1 a sys:Entity, ex:Person ;
  ex:Person.address [
    a ex:Address ;
    ex:Address.city "Richmond"^^xsd:string
  ] ;
  ex:Person.postalAddress [
    a ex:PostalAddress ;
    ex:Address.city "Richmond"^^xsd:string
  ] .
```

If the nested instance has its own Xeto id, that id becomes a named RDF
individual instead of a blank node.

```xeto
@person1: Person {
  address @address1: Address {
    city: "Richmond"
  }
}
```

```turtle
ex:person1 ex:Person.address ex:address1 .

ex:address1 a sys:Entity, ex:Address ;
  ex:Address.city "Richmond"^^xsd:string .
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

## Lists

### SHACL Validation

A Xeto `List` is one ordered collection, not several independent values for the
same slot. RDF represents that collection as a linked list: each node stores
one item in `rdf:first` and points to the remainder of the list through
`rdf:rest`. The exporter checks each `rdf:first` value against the Xeto `of`
type. It counts the linked-list nodes separately so repeated values still count
as separate list items when enforcing `minSize` and `maxSize`.

```xeto
Batch : Dict {
  names: List <of:Str, minSize:2, maxSize:3>
}
```

```turtle
ex:Batch a sh:NodeShape ;
  sh:targetClass ex:Batch ;
  sh:property [
    sh:path ex:Batch.names ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:node [
      sh:property [
        sh:path ( [ sh:zeroOrMorePath rdf:rest ] rdf:first ) ;
        sh:datatype xsd:string ;
      ] ;
      sh:property [
        sh:path [ sh:zeroOrMorePath rdf:rest ] ;
        sh:minCount 3 ;
        sh:maxCount 4 ;
      ]
    ] ;
  ] .
```

An RDF list containing `n` items has `n + 1` nodes because the chain ends at
the additional `rdf:nil` node. Xeto's `minSize` and `maxSize` item counts are
therefore increased by one when converted to these SHACL node counts. In the
example above, `minSize:2` becomes `sh:minCount 3`, and `maxSize:3` becomes
`sh:maxCount 4`.

`nonEmpty` requires at least one list item. Its SHACL node count is two: one
item node plus the terminating `rdf:nil` node.

```xeto
Batch : Dict {
  tags: List <of:Str, nonEmpty>
}
```

```turtle
ex:Batch a sh:NodeShape ;
  sh:targetClass ex:Batch ;
  sh:property [
    sh:path ex:Batch.tags ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:node [
      sh:property [
        sh:path ( [ sh:zeroOrMorePath rdf:rest ] rdf:first ) ;
        sh:datatype xsd:string ;
      ] ;
      sh:property [
        sh:path [ sh:zeroOrMorePath rdf:rest ] ;
        sh:minCount 2 ;
      ]
    ] ;
  ] .
```

When `of` names a dictionary spec, every list item must be an instance of that
spec, so the exporter uses `sh:class`. Scalar item types instead use
`sh:datatype`.

Lists of enum values, Choice selections, units, references, or other lists are
not supported in this version. Exporting one of these list forms is an error;
see [Deferred Mappings](#deferred-mappings).

### Instance Data

A Xeto list becomes an RDF collection, preserving item order.

```xeto
@batch1: Batch {
  names: { "alpha", "beta" }
}
```

```turtle
ex:batch1 a sys:Entity, ex:Batch ;
  ex:Batch.names ("alpha"^^xsd:string "beta"^^xsd:string) .
```

The parenthesized Turtle syntax is shorthand for an `rdf:first`/`rdf:rest`
chain. The same collection can be written out as:

```turtle
ex:batch1 ex:Batch.names _:names1 .

_:names1 rdf:first "alpha"^^xsd:string ;
  rdf:rest _:names2 .

_:names2 rdf:first "beta"^^xsd:string ;
  rdf:rest rdf:nil .
```

The final `rdf:rest rdf:nil` is implicit in the parenthesized form. A malformed
chain that never reaches `rdf:nil` is not a well-formed RDF list.

## Units and Quantities

### Unit and Quantity Correspondence

Xeto defines `sys::Unit` from the standard Project Haystack unit database. Each
unit has a full name and may have one or more symbol aliases. For example,
`square_meter` has the symbol `m²`, while `hour` has both `hr` and `h`. When a
unit has multiple symbols, the last symbol in the standard database is its
preferred display form.

The complete Xeto unit catalog is defined by the
[Xeto `sys::Unit` source](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys/units.xeto).

This specification classifies every unit and quantity in the current Xeto
catalog through machine-readable properties files:

- [Xeto unit to QUDT unit or currency](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-units.props)
- [Xeto quantity to QUDT quantity kind](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-quantities.props)
- [Xeto units without a QUDT mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-unmapped-units.props)
- [Xeto quantities without a QUDT mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-unmapped-quantities.props)

The QUDT release, source graphs, revision, and namespace IRIs used by these
tables are pinned in the
[Xeto-to-QUDT mapping metadata](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-meta.props).
Changing that file is an explicit mapping-version upgrade.

Mapped rows have the form `xetoName=qudtLocalName`. A quantity may list more
than one comma-separated QUDT quantity kind when QUDT uses distinct kinds for
units that Xeto groups together. Unmapped rows use
`xetoName=reasonCode|reviewNote`. The fixed QUDT namespaces are
`http://qudt.org/vocab/unit/` for physical units,
`http://qudt.org/vocab/currency/` for currencies, and
`http://qudt.org/vocab/quantitykind/` for quantity kinds. For example:

```properties
fahrenheit=DEG_F
us_dollar=USD
temperature=Temperature
velocity=Velocity,Speed
belarussian_ruble=noEquivalent|QUDT 2.1 currency graph does not contain obsolete ISO 4217 code BYR
```

The Xeto unit's quantity determines the target namespace. Thus
`fahrenheit=DEG_F` maps to `unit:DEG_F`, while the Xeto currency unit
`us_dollar=USD` maps to `currency:USD`. QUDT types currency resources as
`qudt:CurrencyUnit`, a subclass of `qudt:Unit`.

Xeto units can be written using their full names or any symbol defined for the
same unit. The unit mapping properties file contains one entry per unit, keyed
by its full Xeto name. Before applying the mapping, an implementation resolves
any symbol to the full Xeto unit name, then maps it to the corresponding QUDT
unit. A Xeto `quantity` constraint is looked up by its `sys::UnitQuantity` name
in the quantity-kind mapping.

The mapping preserves the original unit and numeric value. It does not convert
a number into another unit. For example, `32°F` maps to the value `32` with
unit `unit:DEG_F`; it is not converted to `0°C`.

Every current Xeto unit and quantity appears in exactly one mapped or unmapped
inventory. An exporter does not guess a QUDT resource from a similar name or
symbol. A value classified as unmapped is an export error; its inventory row
records why no correspondence is defined.

QUDT publishes its unit definitions in the
[QUDT Units Vocabulary](https://www.qudt.org/doc/DOC_VOCAB-UNITS.html) and
provides downloadable vocabulary graphs through the
[QUDT catalog](https://www.qudt.org/catalog/qudt-catalog.html).

### SHACL Validation

A `Unit` slot contains a canonical QUDT unit IRI. The shape requires
`qudt:Unit`. If the slot specifies a quantity, the unit's QUDT quantity kind
must match it directly or through the hierarchy described below. The validator
reads those relationships from the separately loaded QUDT vocabulary graph.

```xeto
Sensor : Dict {
  unit: Unit <quantity:"temperature">
}
```

```turtle
ex:Sensor a sh:NodeShape ;
  sh:targetClass ex:Sensor ;
  sh:property [
    sh:path ex:Sensor.unit ;
    sh:class qudt:Unit ;
    sh:node [
      sh:property [
        sh:path (
          qudt:hasQuantityKind
          [ sh:zeroOrMorePath skos:broader ]
        ) ;
        sh:hasValue quantitykind:Temperature ;
      ]
    ] ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

#### Quantity Kind Hierarchy

The standard Xeto unit catalog groups each unit under a quantity. For example,
`gallon`, `fluid_ounce`, and `imperial_gallon` are in Xeto's `volume` group.
The quantity-kind mapping relates that Xeto quantity to
`quantitykind:Volume`:

```properties
volume=Volume
```

QUDT gives some units a more specific quantity kind. In the pinned QUDT graph,
the US gallon is classified as `LiquidVolume`, and QUDT identifies
`LiquidVolume` as a narrower concept beneath `Volume`:

```turtle
unit:GAL_US qudt:hasQuantityKind quantitykind:LiquidVolume .

quantitykind:LiquidVolume
  skos:broader quantitykind:Volume .
```

For this mapping, a QUDT quantity kind satisfies a Xeto quantity constraint
when it is the mapped kind or when following one or more QUDT `skos:broader`
relationships reaches the mapped kind. Therefore `unit:GAL_US` satisfies
Xeto's `volume` constraint. Checking only the unit's direct QUDT quantity kind
would incorrectly reject it.

This is a rule of this specification, not a requirement imposed on every QUDT
consumer. The SHACL `sh:zeroOrMorePath` performs the hierarchy traversal
directly, so validation does not require a SKOS reasoner. The separately loaded
QUDT graph must provide both the units' `qudt:hasQuantityKind` statements and
the quantity kinds' `skos:broader` statements. `skos:broader` is defined by the
[W3C SKOS recommendation](https://www.w3.org/TR/skos-reference/#semantic-relations).

Only Xeto units listed in the
[unit mapping](https://github.com/Project-Haystack/xeto/blob/master/src/xeto/sys.rdf/qudt-units.props)
can be exported. Encountering a unit in the unsupported inventory is an export
error.

A number constrained by `unit` or `quantity` is represented as a
`qudt:QuantityValue` node rather than as a plain numeric literal. Numeric range
constraints apply to `qudt:numericValue`; unit constraints apply to
`qudt:unit`.

```xeto
Meter : Dict {
  percent: Number <minVal:0, maxVal:100, unit:"%">
  power: Number <quantity:"power">
}
```

The `percent` shape is:

```turtle
ex:Meter a sh:NodeShape ;
  sh:targetClass ex:Meter ;
  sh:property [
    sh:path ex:Meter.percent ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
    sh:node [
      sh:class qudt:QuantityValue ;
      sh:property [
        sh:path qudt:numericValue ;
        sh:datatype xsd:double ;
        sh:minInclusive "0.0"^^xsd:double ;
        sh:maxInclusive "100.0"^^xsd:double ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
      ] ;
      sh:property [
        sh:path qudt:unit ;
        sh:class qudt:Unit ;
        sh:hasValue unit:PERCENT ;
        sh:minCount 1 ;
        sh:maxCount 1 ;
      ]
    ] ;
  ] .
```

The `power` shape has the same structure but replaces the exact unit with a
`qudt:hasQuantityKind quantitykind:Power` constraint.

### Instance Data

A `Unit` value becomes its canonical QUDT unit IRI:

```turtle
ex:sensor1 ex:Sensor.unit unit:DEG_F .
```

A unit-bearing number becomes a `qudt:QuantityValue` node:

```xeto
@meter1: Meter {
  percent: 50%
  power: 1kW
}
```

```turtle
ex:meter1 a sys:Entity, ex:Meter ;
  ex:Meter.percent [
    a qudt:QuantityValue ;
    qudt:numericValue "50.0"^^xsd:double ;
    qudt:unit unit:PERCENT
  ] ;
  ex:Meter.power [
    a qudt:QuantityValue ;
    qudt:numericValue "1.0"^^xsd:double ;
    qudt:unit unit:KiloW
  ] .
```

Currency units follow the same Number rule:

```xeto
Invoice : Dict {
  price: Number <unit:"\$">
}

@invoice1: Invoice {
  price: 19.99$
}
```

```turtle
ex:invoice1 ex:Invoice.price [
  a qudt:QuantityValue ;
  qudt:numericValue "19.99"^^xsd:double ;
  qudt:unit currency:USD
] .
```

The exporter does not round currency to a fixed number of decimal places or
claim exact decimal arithmetic that the Xeto value does not provide. An
application that requires accounting semantics must define its own rounding
or integer-minor-unit convention before RDF export.

The QUDT declarations and quantity-kind facts referenced by these values are
loaded separately during validation as described under
[Metadata and External Vocabularies](#metadata-and-external-vocabularies).

## Standalone UnitQuantity Values

A standalone `UnitQuantity` value maps to `sys:UnitQuantity.<key>`, using the
exact, case-sensitive enum key and the versioned `sys` namespace. One Xeto
quantity can correspond to several QUDT quantity kinds, or have no QUDT match.
Giving it its own resource preserves one Xeto value without choosing a single
QUDT match or turning it into several values.

```xeto
Reading : Dict {
  quantity: UnitQuantity
}

@reading1: Reading { quantity: "energy" }
```

```turtle
ex:reading1 a sys:Entity, ex:Reading ;
  ex:Reading.quantity sys:UnitQuantity.energy .

sys:UnitQuantity.energy a sys:UnitQuantity ;
  rdfs:label "energy"@en ;
  rdfs:seeAlso quantitykind:Energy, quantitykind:MomentOfForce,
    quantitykind:Torque .
```

Each quantity resource has a label and `rdfs:seeAlso` links to its matching
QUDT quantity kinds, of which there may be several. For example, Xeto's
`energy` category maps to both `quantitykind:Energy` and `quantitykind:Torque`.
These links do not assert equivalence: the link to `quantitykind:Torque` does
not mean that every instance with quantity `energy` measures torque. A quantity without
a QUDT match, such as `powerByVolumetricFlow`, has its own resource and label,
but no QUDT links.

All standalone `UnitQuantity` slots reference a shared
`sys:UnitQuantityShape`:

```turtle
sys:UnitQuantityShape a sh:NodeShape ;
  sh:nodeKind sh:IRI ;
  sh:in (sys:UnitQuantity.energy sys:UnitQuantity.temperature # ...
  ) .
```

Each exported schema graph containing a `UnitQuantity` slot includes this
shape once. It requires the value to be an IRI from the quantity catalog.
The `sh:in` list contains a resource for every effective `sys::UnitQuantity`
enum entry, using the versioned `sys` namespace. The example above abbreviates
that list.

The property shape for each standalone `UnitQuantity` slot references the
shared shape and defines that slot's cardinality and invariant constraints.
For example, `Reading.quantity`
requires exactly one quantity value:

```turtle
ex:Reading a sh:NodeShape ;
  sh:targetClass ex:Reading ;
  sh:property [
    sh:path ex:Reading.quantity ;
    sh:node sys:UnitQuantityShape ;
    sh:minCount 1 ;
    sh:maxCount 1
  ] .
```

An optional slot omits `sh:minCount`. An invariant adds, for example,
`sh:hasValue sys:UnitQuantity.energy`.

## Queries

A Xeto `Query` is computed navigation, not a stored required field. It selects
records by following references from a starting record. This section calls a
SPARQL query that performs that selection over exported RDF data a
**SPARQL selection query**. The term Query refers to the Xeto definition;
SPARQL selection query refers to its SPARQL translation.

Query traversal uses the [reference-field metadata](#reference-field-metadata)
in exported instance data to distinguish Refs from other IRI-valued fields.

### SHACL Validation

Query slots without named body requirements impose no validation constraints. Their
`via` or `inverse` metadata is retained as a SHACL property path, without
`sh:class`, `sh:minCount`, or `sh:maxCount`. The `of` type selects query results;
it does not require every record along the path to have that type.

#### Supported Query Paths

```xeto
Equip : Dict {
  plainPoints: Query<of:Point>
  points: Query<of:Point, inverse:"Point.equips">
}

Point : Dict {
  equipRef: Ref<of:Equip>
  equips: Query<of:Equip, via:"equipRef+">
}

Node : Dict {
  next: Ref<of:Node, maybe>
  reachable: Query<of:Node, via:"next*">
}
```

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    sh:path ex:Equip.plainPoints ;
  ] ;
  sh:property [
    sh:path [
      sh:oneOrMorePath [ sh:inversePath ex:Point.equipRef ]
    ] ;
  ] .

ex:Point a sh:NodeShape ;
  sh:targetClass ex:Point ;
  sh:property [
    sh:path ex:Point.equipRef ;
    sh:class ex:Equip ;
    sh:nodeKind sh:IRI ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path [ sh:oneOrMorePath ex:Point.equipRef ] ;
  ] .

ex:Node a sh:NodeShape ;
  sh:targetClass ex:Node ;
  sh:property [
    sh:path ex:Node.next ;
    sh:class ex:Node ;
    sh:nodeKind sh:IRI ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path [ sh:zeroOrMorePath ex:Node.next ] ;
  ] .
```

A Query without named body requirements does not require a particular number
of matches. Its path may reach no records, one record, or many records. Only
records that fit the Query's `of` type are returned; reaching other types does
not invalidate the record containing the Query. A `+` path follows one or more
references and may pass through records that do not fit `of`. Cycles are allowed,
and each matching record is returned once.

The Query's SHACL property path imposes no class or cardinality constraint.
Constraints on the stored Ref or MultiRef remain unchanged: `Ref<of:Point>`
still requires its referenced record to have class Point. Query execution and
validation of the returned records are separate operations.

Given only this instance relationship:

```turtle
ex:point1 ex:Point.equipRef ex:equip1 .
```

the inverse path can navigate from `ex:equip1` to `ex:point1`. The instance
graph does not need this materialized reverse triple:

```turtle
ex:equip1 ex:Equip.points ex:point1 .
```

A plain query without `via` or `inverse` uses its slot property as `sh:path`,
but still has no cardinality constraint. Required and optional members declared inside a
query body are not mapped in this version; see
[Deferred Mappings](#deferred-mappings).

`via` and `inverse` are mutually exclusive. A `via` path is an unqualified
slot name, a `Spec.slot` name, or a fully qualified `lib::Spec.slot` name,
optionally followed by `+` or `*`. An `inverse` reference uses the same name
forms without a path suffix. Empty paths, malformed names, repeated suffixes,
and a Query declaring both forms are export errors.

#### Rejected Query Metadata

The following complete definitions are rejected during RDF export. Declaring
both traversal forms is ambiguous:

```xeto
Point : Dict
Equip : Dict {
  points: Query<of:Point, via:"pointRef", inverse:"Point.equips">
}
```

The exporter reports that `fixture.example::Equip.points` cannot declare both
`via` and `inverse`.

An empty traversal does not identify a property:

```xeto
Point : Dict
Equip : Dict {
  points: Query<of:Point, via:"">
}
```

The exporter reports `Empty query path for fixture.example::Equip.points`.
The same rule applies to an empty `inverse` value.

A path may have at most one trailing quantifier:

```xeto
Point : Dict
Equip : Dict {
  points: Query<of:Point, via:"pointRef++">
}
```

The exporter reports an invalid Query path for
`fixture.example::Equip.points`, including the rejected value `pointRef++`.

#### Inverse Query and Direct Inverse Property

An inverse reference is resolved in the library that declares the Query. If
the referenced name identifies another Query, that Query must declare `via`;
the exporter reverses its `via` path. If the name identifies an ordinary
property, or no Query declaration with that name is present, the exporter
reverses the named property directly. These two cases are deliberately
distinct:

```xeto
Equip : Dict {
  // Reverses the traversal declared by Point.equips.
  points: Query<of:Point, inverse:"Point.equips">

  // Reverses Point.equipRef directly.
  directPoints: Query<of:Point, inverse:"Point.equipRef">
}

Point : Dict {
  equipRef: Ref<of:Equip>
  equips: Query<of:Equip, via:"equipRef+">
}
```

```turtle
ex:Equip a sh:NodeShape ;
  sh:targetClass ex:Equip ;
  sh:property [
    sh:path [
      sh:oneOrMorePath [ sh:inversePath ex:Point.equipRef ]
    ] ;
  ] ;
  sh:property [
    sh:path [ sh:inversePath ex:Point.equipRef ] ;
  ] .
```

`Equip.points` reverses the referenced Query's `equipRef+` traversal, so its
path retains `sh:oneOrMorePath`. `Equip.directPoints` reverses the ordinary
`Point.equipRef` property directly, so its path has no repetition operator.

A referenced Query without `via` does not contain enough traversal metadata
to invert and is therefore an export error. It is not silently treated as an
ordinary stored property.

### Queries Contributed by Mixins

A query contributed by a mixin may traverse a slot from the contributing
library or a slot from the target spec's library. The path name determines
which RDF predicate SHACL follows.

**Example: `mixin-query-paths`.**

Suppose `base.equips` defines:

```xeto
Equip : Dict {
  basePointRef: Ref<of:Point, maybe>
}

Point : Dict
```

and `app.equips` contributes:

```xeto
+base.equips::Equip {
  pointRef: Ref<of:base.equips::Point, maybe>
  pointsVia: Query<of:base.equips::Point, via:"pointRef">
  pointsViaBase: Query<of:base.equips::Point,
    via:"base.equips::Equip.basePointRef">
  pointsInverse: Query<of:base.equips::Point, inverse:"Point.equips">
}

+base.equips::Point {
  equipRef: Ref<of:base.equips::Equip, maybe>
  equips: Query<of:base.equips::Equip, via:"equipRef">
}
```

The relevant part of the resulting target shape is:

```turtle
base:Equip a sh:NodeShape ;
  sh:targetClass base:Equip ;
  sh:property [
    sh:path app:Equip.pointRef ;
  ] ;
  sh:property [
    sh:path base:Equip.basePointRef ;
  ] ;
  sh:property [
    sh:path [ sh:inversePath app:Point.equipRef ] ;
  ] .
```

For `pointsVia`, the unqualified `pointRef` resolves in the contributing
`app.equips` library, producing `app:Equip.pointRef`. For `pointsViaBase`, the
full name explicitly selects `base:Equip.basePointRef`. This determines which
RDF relationships SHACL follows; the two paths are not interchangeable.

For `pointsInverse`, `Point.equips` first resolves to the query contributed by
`app.equips`. That query traverses `app:Point.equipRef`, so reversing it
produces `[ sh:inversePath app:Point.equipRef ]`.

### Query Execution and Result Validation

An executable translation has three parts: SHACL checks for inputs that would
cause a query execution error, a SPARQL selection query, and SHACL constraints for
separately validating the selected records. The property-path metadata
above does not perform these operations by itself.

Query selection and returned-record validation are separate operations. For
example:

```xeto
Target: Dict { hit }
Holder: Dict {
  next: MultiRef?
  direct: Query<of:Target, via:"next">
}
@a: Holder { next: MultiRef { @t } }
@t: Target { hit }
```

The Query `direct` starting at `a` returns `t`. Validation against `Target`
checks that `t` has `hit`. Nominal membership in Target does not establish
that its required fields are present.

The corresponding instance RDF is:

```turtle
ex:a a sys:Entity, ex:Holder ;
  ex:Holder.next ex:t ;
  sys:hasRefField ex:Holder.next .
ex:t a sys:Entity, ex:Target ;
  sys:hasMarker ex:Target.hit .
```

Here, `ex:` and `sys:` denote the concrete library-version namespaces.
The following fragments show only `ex:Holder.next` and `ex:Target`. A complete
translation must preserve two separate rules:

- **Follow the field by name.** `via:"next"` reads the starting record's `next`
  field. `via:"next+"` also reads `next` on reached records. Different specs may
  map that field name to different RDF property IRIs; the SPARQL path must
  recognize all of them.
- **Include result subtypes.** The SPARQL selection query must accept records
  of type `Target` or any of its subtypes.

For this one-hop example, traversal follows `ex:Holder.next` from `ex:a`.
For `via:"next+"`, traversal continues through each reached record's `next`
field, which may use a different property IRI. Including those IRIs does not
select unrelated records; the path still starts at the supplied starting record.

#### Query Input Validation

Input SHACL requires the starting record to exist and populated fields followed
by the Query to contain references. These checks must pass before the SELECT
is executed. For the starting record `a`:

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:a ;
  sh:nodeKind sh:IRI ;
  sh:property [ sh:path rdf:type ; sh:hasValue sys:Entity ] ;
  sh:or (
    [ sh:property [ sh:path ex:Holder.next ; sh:maxCount 0 ] ]
    [ sh:property [ sh:path sys:hasRefField ; sh:hasValue ex:Holder.next ] ]
  ) ;
  sh:property [ sh:path ex:Holder.next ; sh:nodeKind sh:IRI ] ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:not [ sh:in ( ex:Holder.next ) ]
  ] .
```

The existence check requires an exported record at `ex:a`. The remaining checks
allow an absent field, require reference-field metadata for a populated field,
and reject literal, blank-node, or Marker values. A failed input check is an
execution error; the consumer must not turn it into an empty query result.

The starting record must exist. If the supplied starting ID has no record,
execution must report an error, rather than return an empty result. This differs
from a missing referenced target, which ends only that traversal branch:

| Input | Correct behavior | Incorrect translation |
| --- | --- | --- |
| Starting record `a` is missing | Report an execution error. | Return no results. |
| `a.next` is a Ref to an existing `t: Target` | Return `t`. | Drop `t`. |
| `a.next` is a Ref to a missing record | End that branch without a result. | Report an execution error. |
| `a.next` stores a Unit or named nested Dict | Report an error when trying to follow it. | Follow its IRI or silently return no results. |
| A Ref reaches `t: Target`, but `t` lacks `hit` | Return `t`; report its validation failure separately if validation is requested. | Remove `t` from the results. |

The [reference-field rules](#reference-field-metadata) govern whether populated
fields can be followed; the [Query path rules](#supported-query-paths) govern
which records are selected. For forward `via+` traversal, reference checks apply wherever
traversal continues; unreachable records are outside those checks. The
[inverse traversal rules](#inverse-result-selection) specify the wider scope
for transitive inverse queries. Source-type and declared Ref
cardinality constraints must not be added as query execution requirements.

#### SPARQL Result Selection

A SPARQL selection query follows the Query's reference-field path from the
starting record and returns distinct matching records. Each result must be an IRI
identifying an exported Entity and must fit the Query's `of` type. For a
nominal target this is type membership; sugar also permits the structural
matching described below. Results have no guaranteed order.

Traversal may be forward or inverse, and may follow one hop or one or more
hops. The result-type constraint applies only to the final record, not to
intermediate records. Inverse traversal does not apply the referenced Query's
result constraints to intermediate records either.

The reference-field tag followed by the path can map to one or several
property IRIs:

- A single property uses its IRI directly.
- Multiple properties use a parenthesized `|` alternative.
- Inverse traversal reverses the property path with `^`.
- One-hop traversal uses the forward or inverse path directly; one-or-more
  traversal appends `+` to that path.

The property path includes all IRIs representing the traversed reference-field
tag across the effective specs in the current namespace, using the declared-slot
and undeclared-field mapping rules. This set is independent of the eligible
result classes.

For `Holder.direct` in the
[Query execution example](#query-execution-and-result-validation), the SPARQL
selection query binds the starting record `ex:a`, follows `ex:Holder.next`, and
selects records of type `ex:Target`:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:a }
  ?subject ex:Holder.next ?result .
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Target }
  }
}
```

This returns `ex:t`. `VALUES` binds the starting record, the property pattern
follows its reference, and the type pattern selects Target records. `DISTINCT`
returns each identity once; the query promises no order. A missing reference
target has no exported Entity record and produces no result.

#### Sugar and Compound Result Selection

Sugar targets allow type membership or structural matching against an
underlying named type. Union and intersection targets use type membership.
Result selection does not perform full record validation.

##### Sugar Matching

For example, consider these definitions. The illustrative library name is
`example.fitting`, and `ex:` denotes its versioned RDF namespace:

```xeto
Base: Dict { required, *hot: Marker, *extra: Marker, *label: Str }
Sweet: Base <sugar> { hot, label: "default" }
Queries: Dict {
  sweet: Query<of:Sweet, via:"id">
}
@candidate: Base { required, hot }
```

The relevant instance RDF is:

```turtle
ex:candidate a sys:Entity, ex:Base ;
  sys:hasMarker ex:Base.required, ex:Base.hot .
```

`candidate` is declared as `Base`, not `Sweet`. It still matches `of:Sweet`
because its type matches Sweet's underlying named type, or **nominal anchor**,
`Base`, and it has the required `hot` field. The default `label` value is not
a selection requirement.

The `via:"id"` path selects the starting record itself. Its SPARQL translation
uses `BIND(?subject AS ?result)` rather than following a stored property.
The result predicate has two alternatives:

- The record's type fits `Sweet`.
- The record's type fits `Base`, and its `hot` field is present.

Class and property lists are abbreviated to the relevant definitions. A
translation must include all applicable alternatives in the current namespace.
The SPARQL selection query is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:candidate }
  BIND(?subject AS ?result)
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER(
    EXISTS { ?result a ?type . VALUES ?type { ex:Sweet } }
    || (
      EXISTS { ?result a ?anchor . VALUES ?anchor { ex:Base ex:Sweet } }
      && (
        EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot }
          ?result ?property ?value .
        }
        || EXISTS {
          VALUES ?property { ex:Base.hot ex:Sweet.hot }
          ?result sys:hasMarker ?property .
        }
      )
    )
  )
}
```

The second alternative selects `ex:candidate`. The two presence tests recognize
a field stored as an ordinary value or as a Marker, respectively.

##### Selection Is Not Validation

A runtime record with `hot` but no `required` marker has this relevant RDF:

```turtle
ex:candidate a sys:Entity, ex:Base ;
  sys:hasMarker ex:Base.hot .
```

The SPARQL selection query is unchanged:

| Operation | Outcome |
| --- | --- |
| Select with `of:Sweet` | Return `candidate`: `hot` is still present. |
| Validate against `Sweet` | Fail: the inherited `required` marker is missing. |

Sugar's structural alternative checks its required markers and invariant
values, including requirements inherited from other sugar definitions. It does
not check defaults or ordinary required fields inherited from its nominal
anchor. A required marker checks field presence, not the stored value's type.
Returned-record validation checks those additional requirements.

Runtime records can be missing required fields. Each table row represents a
separate starting record with an `id`; the other fields use Trio notation,
separated by semicolons.

| Runtime fields | Selected by `sweet`? | Valid against Sweet? |
| --- | --- | --- |
| `spec: @example.fitting::Base`; `required`; `hot` | Yes | Yes |
| `spec: @example.fitting::Base`; `hot` | Yes | No: `required` is missing |
| `spec: @example.fitting::Base`; `required`; `hot: "oops"` | Yes | No: `hot` is not a Marker |
| `spec: @example.fitting::Base`; `required`; `hot`; `label: "different"` | Yes | Yes: the default does not require this value to equal `"default"` |
| `spec: @example.fitting::Base`; `required` | No | No: `hot` is missing |
| `spec: @example.fitting::Sweet`; `required` | Yes: its declared type matches | No: `hot` is missing |

##### Computed Sugar Subtyping

A sugar type with the same anchor and additional requirements can fit `Sweet`
without a declared inheritance link:

```xeto
MoreSweet: Base <sugar> { hot, extra }
```

`MoreSweet` does not declare `Sweet` as its base. It nevertheless fits `Sweet`:
both have the anchor `Base`, and MoreSweet's requirements include Sweet's
required `hot` marker. The `label` default does not participate in this comparison.

The type-membership alternative for `of:Sweet` must therefore include both
classes:

```sparql
EXISTS {
  ?result a ?type .
  VALUES ?type { ex:Sweet ex:MoreSweet }
}
```

For example, this runtime record declares `MoreSweet` but has no markers:

```trio
id: @example.fitting::typedCandidate
spec: @example.fitting::MoreSweet
```

Its relevant RDF is:

```turtle
ex:typedCandidate a sys:Entity, ex:MoreSweet .
```

It matches the type-membership alternative for `of:Sweet`, then fails
returned-record validation because the required markers are missing. In
general, computed sugar subtyping requires a compatible anchor and every target
requirement with the same required value; the candidate may add requirements.

##### Invariant Values

An invariant, unlike a default, adds a value-equality test to sugar's structural
alternative. For example:

```xeto
Named: Base <sugar> {
  label: Str <invariant> "wanted"
}
```

For records declared as `Base`, the structural alternative for `of:Named`
requires `label` to equal `"wanted"`:

```sparql
EXISTS {
  VALUES ?property { ex:Base.label ex:Named.label }
  ?result ?property ?label .
  FILTER(?label = "wanted")
}
```

This fragment supplies the value test alongside the anchor-membership test.
As with `Sweet`, a declared type that fits `Named` can match the separate
type-membership alternative without running the structural tests.

Each structural test must recognize all property IRIs representing its field
name, including inherited and undeclared-field mappings. Invariant comparisons
must preserve Xeto value equality, including supported full-fidelity and
Haystack representations; RDF lexical equality alone is insufficient. A
translation must reject an invariant whose equality it cannot preserve rather
than omit the requirement.

##### Union Targets

A union target can include a sugar type:

```xeto
Other: Dict { other }
Either: Sweet | Other
CompoundQueries: Dict {
  either: Query<of:Either, via:"id">
}
```

With `candidate` declared as `Base` and containing `required` and `hot`, the
two Query result types produce different selections:

| Query result type | Selected? |
| --- | --- |
| `Sweet` | Yes: Base membership and the `hot` field satisfy sugar's structural alternative. |
| `Either` | No: Base does not fit either union member by type membership. |

A union target checks type membership in its alternatives. It does not apply
each member's structural sugar alternative. The identity binding, Entity check,
and IRI check stay the same; only the result predicate changes:

```sparql
FILTER EXISTS {
  ?result a ?type .
  VALUES ?type { ex:Sweet ex:MoreSweet ex:Other ex:Either }
}
```

The list includes computed sugar subtypes such as `MoreSweet`, but no structural
`hot` test. Other qualifying types in the current namespace must also be included.

##### Intersection Targets

An intersection also selects by type membership, not by checking whether a
record happens to have all of its members' fields. For example:

```xeto
Both: Base & Other
IntersectionQueries: Dict {
  both: Query<of:Both, via:"id">
}
```

These runtime records have the same marker names but different declared types:

```trio
id: @example.fitting::baseRecord
spec: @example.fitting::Base
required
other
---
id: @example.fitting::bothRecord
spec: @example.fitting::Both
required
other
```

Their relevant RDF is:

```turtle
ex:baseRecord a sys:Entity, ex:Base ;
  sys:hasMarker ex:Base.required, ex:Base.other .
ex:bothRecord a sys:Entity, ex:Both ;
  sys:hasMarker ex:Base.required, ex:Other.other .
```

| Starting record | Selected by `both`? |
| --- | --- |
| `baseRecord` | No: having both fields does not make its declared type fit Both. |
| `bothRecord` | Yes: its declared type is Both. |

The result predicate tests the eligible classes, not the fields:

```sparql
FILTER EXISTS {
  ?result a ?type .
  VALUES ?type { ex:Both }
}
```

Compound source types also participate in membership. `Both` fits `Base`
through its Base member, and fits `Either` through its Other member. `Either`
does not fit `Base`, because its Other alternative does not fit Base. Thus an
intersection source can match through a member, while a union source must
match through all members. Ordinary inheritance still applies.

A named subtype of a union does not itself acquire the union target's
member-selection rule. Nested compounds use the same membership rules. Class
alternatives must be computed from these relationships, not inferred merely
from the fields on a record.

##### Other Result Types

Choice targets also use type membership; missing member markers are validation
failures rather than selection failures. Scalar targets do not match ordinary
records. Malformed runtime records can assert a scalar type and pass the
initial type-membership check. A translator that does not support the resulting
scalar conformance checks must reject that Query rather than assume its result
is always empty.

#### Returned-Record Validation

Returned-record validation applies the Query's `of` constraints only to the
selected identities. A selection shape uses `sh:targetNode` for each returned
IRI and `sh:node` for reusable constraints without automatic targets. For the
result `ex:t`:

```turtle
ex:SelectedResults a sh:NodeShape ;
  sh:targetNode ex:t ;
  sh:node ex:TargetResultConstraints .

ex:TargetResultConstraints a sh:NodeShape ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:hasValue ex:Target.hit
  ] .
```

Target requires only the `hit` marker, represented here by `sh:hasValue`.
The reusable constraints must not independently target unrelated records.
The full instance graph and schema vocabulary remain available during
validation so constraints on references can inspect the referenced records.

With the RDF above, result validation passes. Removing
`ex:t sys:hasMarker ex:Target.hit` leaves the query result unchanged but makes
result validation fail. Selecting a record and validating its contents thus
remain separate, as they are in Xeto.

A translation must use the same resolved library versions as the RDF export.
It must reject query forms for which it cannot preserve these behaviors rather
than silently omit traversal or validation rules.

##### Validating Sugar Results

A sugar result can be declared as its nominal anchor rather than the sugar
type. Validation must recognize the field IRIs used by those records. For
`candidate` in [Sugar Matching](#sugar-matching), that means recognizing Base's
field IRIs as well as Sweet's.

Checking only `ex:Sweet.hot` would miss the candidate's
`sys:hasMarker ex:Base.hot` statement. The required-marker checks accept the
relevant `hot` marker IRIs:

```turtle
ex:SelectedSugarResults a sh:NodeShape ;
  sh:targetNode ex:candidate ;
  sh:node ex:SweetResultConstraints .

ex:SweetResultConstraints a sh:NodeShape ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:hasValue ex:Base.required
  ] ;
  sh:property [
    sh:path sys:hasMarker ;
    sh:qualifiedValueShape [
      sh:in ( ex:Base.hot ex:Sweet.hot ex:MoreSweet.hot )
    ] ;
    sh:qualifiedMinCount 1
  ] .
```

The `candidate` record with both `required` and `hot` passes these marker checks. Removing `required` causes
validation to fail without changing selection. Replacing the `hot` Marker
with a string also fails validation: the ordinary value establishes presence
for selection, but does not supply the required Marker statement.

The fragment shows only the marker checks. Complete reusable constraints must
recognize all applicable field mappings across the current namespace and check
required fields, value types, and invariants, including inherited requirements.
Ordinary value-bearing fields use alternative property paths; marker checks
accept alternative marker IRIs on `sys:hasMarker`. This changes which RDF
properties represent a field, not the requirements that field must satisfy.

#### Inverse Result Selection

An inverse Query uses the referenced Query's path in reverse and its own
`of` type to select results. The referenced Query's `of` type does not filter
the starting record or intermediate records. No stored triples for computed
Query slots are required.

For example, in library `example.inverse`:

```xeto
Equipment: Dict {
  points: Query<of:Point, inverse:"example.inverse::Point.equips">
}
Point: Dict {
  point
  equipRef: Ref?
  equips: Query<of:Equipment, via:"equipRef+">
}
Relay: Dict { equipRef: Ref? }
@equip1: Equipment {}
@p1: Point { point, equipRef: @relay1 }
@relay1: Relay { equipRef: @equip1 }
@p2: Point { point, equipRef: @p1 }
```

Starting `points` at `equip1` returns `p1` and `p2`. Both have a nonempty
`equipRef` path to `equip1`; the Relay is traversed but is not a Point result.
The corresponding SELECT uses inverse property alternatives:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:equip1 }
  ?subject (^(ex:Point.equipRef|ex:Relay.equipRef))+ ?result .
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Point }
  }
}
```

As in the forward example, the displayed path includes the properties used by
the instances; a translation includes all properties representing the followed
tag across the effective specs in the current namespace. Inherited and
dependency Query definitions supply their effective path. A missing referenced Query or one without a `via` path
is an error, rather than an empty result.
Executable inverse queries require a fully qualified reference such as
`example.inverse::Point.equips`. Native execution does not resolve an
unqualified inverse name, even though the schema property-path mapping accepts
local names.

An inverse of a one-hop path omits `+`. Native one-hop inverse matching compares
stored values with the starting record's ID without dereferencing them. A
non-Ref value does not match, even if it exports to the same IRI as that record.
The SELECT must therefore require reference-field evidence on a matching edge.
Its existence test binds `?referenceProperty` to the same property set used in
the path, requires the result record's `sys:hasRefField` to name that property,
and requires that property to point to the starting record. These two triples
must use the same property binding; evidence for a different field is not
sufficient.

For a one-hop version of the example's `equips` Query, the complete SELECT
structure is shown below. Its property alternatives are abbreviated to the
properties used by these instances; the full set also includes the other
resolved mappings for `equipRef`, including undeclared fields.

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:equip1 }
  ?subject ^(ex:Point.equipRef|ex:Relay.equipRef) ?result .
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    VALUES ?referenceProperty { ex:Point.equipRef ex:Relay.equipRef }
    ?result sys:hasRefField ?referenceProperty ;
            ?referenceProperty ?subject .
  }
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Point }
  }
}
```

This one-hop query returns no Points in the example: only `relay1` points
directly to `equip1`, and Relay does not match Point. The earlier `+` query
also follows the incoming links to `p1` and `p2` and returns both.

Inverse traversal over ordinary reference fields searches records carrying the
followed tag throughout the input graph. Unlike forward traversal, its search is not limited to records
reachable from the starting record. For an inverse of `via+`, input checks
therefore cover those candidate records even in disconnected components.
Direct inverse matching requires an existing starting record but does not
reject non-Ref fields merely because they are present elsewhere in the graph.

Repeated links and cycles do not duplicate results. The starting record is
eligible only if a nonempty path leads back to it and it fits the inverse
Query's `of` type. Returned-record SHACL uses that same `of` type, independently
of the referenced Query's result constraints.

#### Identity Traversal

The `id` field is a reference to the record itself. RDF represents that value
as the record's subject IRI, not as a property ending in `.id`. An executable
translation of `via:"id"` must therefore select the starting identity directly.
One identity step is a self-link: repeating it with `+`, or reversing it with
`inverse`, reaches the same identity. This is a nonempty native traversal,
not a rule that all one-or-more paths include their starting record.

For example, in library `example.identity`:

```xeto
Target: Dict { hit }
Other: Dict {}
Queries: Dict {
  self: Query<of:Target, via:"id">
  repeatedSelf: Query<of:Target, via:"id+">
  inverseSelf: Query<of:Target, inverse:"example.identity::Queries.self">
  inverseRepeatedSelf: Query<of:Target, inverse:"example.identity::Queries.repeatedSelf">
}
@t: Target { hit }
@outsider: Other {}
```

Each of these Queries returns `t` when started at `t`, and no results when
started at `outsider`. The common SELECT structure substitutes the supplied
starting IRI into `VALUES` and the Query's own eligible result classes into
the result-type test. Its navigation clause is `BIND(?subject AS ?result)`;
it requires no tag-property alternatives or reference-field evidence.
For this example, the complete selection query starting at `t` is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:t }
  BIND(?subject AS ?result)
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Target }
  }
}
```

As with other nominal selection queries, the result-class set includes any
subtypes in the resolved schemas. For an inverse Query it comes from that
Query's own `of`, independently of the referenced Query's `of`.

Input validation must still reject a missing starting record. Identity
traversal does not inspect other fields or disconnected records. Its input
shape for the same start is:

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:t ;
  sh:nodeKind sh:IRI ;
  sh:property [ sh:path rdf:type ; sh:hasValue sys:Entity ] .
```

Result validation remains separate. The returned `t` is checked against
Target using a selection shape, with the complete graph available:

```turtle
ex:SelectedResults a sh:NodeShape ;
  sh:targetNode ex:t ;
  sh:node ex:TargetResultConstraints .
ex:TargetResultConstraints a sh:NodeShape ;
  sh:property [ sh:path sys:hasMarker ; sh:hasValue ex:Target.hit ] .
```

Removing `t`'s `hit` marker does not remove `t` from the query result; it makes
the separate validation fail. A missing starting record instead fails the
input check before selection, even if the SELECT alone would return no rows.

#### Spec-Reference Traversal

The [`spec` field](#the-spec-field) identifies a record's declared Xeto type.
Its Ref can be used in two different operations:

- **Type validation** resolves the Ref to a schema definition and checks the
  record against that definition.
- **Query traversal** reads the Ref and looks for a runtime record with that
  ID. As with other Query paths, that record is returned only if it exists and
  matches the Query's `of` type. The schema definition alone is not a result.

A SPARQL selection query must therefore follow `sys:spec`, which preserves
the exact Ref in the field, rather than `rdf:type`, which may include other
type memberships. The instance must also have `sys:hasRefField sys:spec`.
A record without a stored `spec` field has no `sys:spec` value; type membership
alone must not create one.

For example, these definitions belong to library `example.spec`:

```xeto
Base: Dict { hit }
Child: Base {}
Queries: Dict {
  direct: Query<of:Base, via:"spec">
  transitive: Query<of:Base, via:"spec+">
  directInverse: Query<of:Base, inverse:"example.spec::Queries.direct">
  transitiveInverse: Query<of:Base, inverse:"example.spec::Queries.transitive">
}
```

Runtime records may have IDs that equal type names. These runtime records are
shown in Trio; library instance declarations cannot reuse a type's name:

```trio
id: @example.spec::reading
spec: @example.spec::Child
hit
---
id: @example.spec::Child
spec: @example.spec::Base
hit
```

Their instance RDF is:

```turtle
ex:reading a sys:Entity, ex:Child ;
  sys:spec ex:Child ;
  sys:hasRefField sys:spec ;
  sys:hasMarker ex:Base.hit .
ex:Child a sys:Entity, ex:Base ;
  sys:spec ex:Base ;
  sys:hasRefField sys:spec ;
  sys:hasMarker ex:Base.hit .
```

Starting `direct` at `reading` returns the runtime record named `Child`.
Starting `transitive` there also returns only `Child`: the schema defines
Base, but the instance graph contains no runtime record named Base. A schema
class alone is not a query result. Conversely, inverse traversal from Child
returns reading. Reading's subtype membership makes it eligible for `of:Base`;
it does not create a direct spec reference from reading to Base.

The traversal property set is exactly `sys:spec`. Direction and repetition
compose with this property as follows:

| Query form | SPARQL path |
| --- | --- |
| `via:"spec"` | `sys:spec` |
| `via:"spec+"` | `(sys:spec)+` |
| Inverse of `via:"spec"` | `^sys:spec` |
| Inverse of `via:"spec+"` | `(^sys:spec)+` |

The starting IRI and eligible result classes supply the remaining substitutions
in the [selection structure](#query-execution-and-result-validation). For the
transitive Query above, the complete SELECT is:

```sparql
SELECT DISTINCT ?result WHERE {
  VALUES ?subject { ex:reading }
  ?subject (sys:spec)+ ?result .
  ?result a sys:Entity .
  FILTER(isIRI(?result))
  FILTER EXISTS {
    ?result a ?resultType . VALUES ?resultType { ex:Base ex:Child }
  }
}
```

Reference-field checks use `sys:spec` in place of ordinary tag-property
alternatives. Forward one-hop checks the start; forward `+` also checks reached
records. Direct inverse selection requires reference-field evidence on the
matching edge, and transitive inverse checks all candidate records, as specified
in [Inverse Result Selection](#inverse-result-selection). An absent spec field
ends a branch; a present non-Ref must not silently become an empty forward result.

The input shape for `transitive` starting at reading is:

```turtle
[] a sh:NodeShape ;
  sh:targetNode ex:reading ;
  sh:nodeKind sh:IRI ;
  sh:property [ sh:path rdf:type ; sh:hasValue sys:Entity ] ;
  sh:property [
    sh:path [ sh:zeroOrMorePath sys:spec ] ;
    sh:node [
      sh:or (
        [ sh:property [ sh:path sys:spec ; sh:maxCount 0 ] ]
        [ sh:property [ sh:path sys:hasRefField ; sh:hasValue sys:spec ] ]
      ) ;
      sh:property [ sh:path sys:spec ; sh:nodeKind sh:IRI ] ;
      sh:property [ sh:path sys:hasMarker ; sh:not [ sh:in ( sys:spec ) ] ]
    ]
  ] .
```

The zero-or-more path includes the starting record in input checks. The SELECT
still requires one or more steps. After selection, the returned Child record
is validated against Base independently:

```turtle
ex:SelectedResults a sh:NodeShape ;
  sh:targetNode ex:Child ;
  sh:node ex:BaseResultConstraints .
ex:BaseResultConstraints a sh:NodeShape ;
  sh:property [ sh:path sys:hasMarker ; sh:hasValue ex:Base.hit ] .
```

Selection operates on instance data. Schema descriptions must not supply
runtime field values. For returned-record validation, the complete instance
graph remains available, together with supporting vocabulary. When a runtime
record and a schema class share an IRI, the schema's `rdfs:subClassOf`
statements and `rdf:type rdfs:Class` declaration at that IRI are retained.
The class declaration allows a stored spec Ref to identify its schema target;
it does not supply a runtime field. Other schema
statements remain outside the data graph. For example, the schema's declaration
of Base's required `hit` marker cannot satisfy a missing `hit` on a runtime
record named Base. Removing a returned record's marker therefore still fails
result validation, without removing the record from the SELECT results.

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
[Queries Contributed by Mixins](#queries-contributed-by-mixins).

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

# Deferred Mappings

The following forms do not have a conformance mapping in this version. An
exporter encountering one of them fails instead of silently emitting weaker
RDF or SHACL. The error identifies the unsupported construct and, when it is
available, the affected qualified spec or slot. This specification does not
define error codes or prescribe programming-language error types.

- Required or optional members inside a `Query` body. Their validation requires graph-mode
  query traversal plus qualified SHACL value constraints.
- `List` item types for enums and choices, pending Xeto/Haystack instance
  fidelity fixes.
- `List` item types for units and refs, and nested `List` values, pending a
  complete instance and validation mapping.
- Direct `Scalar` slots, pending a SHACL mapping for the abstract family of
  special and custom representations. Concrete custom scalars use the mapping
  in [Custom Scalar Datatypes](#custom-scalar-datatypes).
- `Grid` and the abstract `Collection` type, pending a complete collection and
  table mapping.
- `Func`, `Interface`, and `Funcs`, whose API and execution semantics are
  outside the current data profiles.
- `sys::This`, `Ref<of:This>`, and `MultiRef<of:This>` slots, pending a
  parent-relative RDF and SHACL mapping.
- Runtime component and function-block constructs from `sys.comp`.

An `abstract` spec still defines a class and constraints inherited by its
concrete descendants. An instance cannot name the abstract spec as its direct
type, so its node shape rejects an `rdf:type` value equal to that abstract
class. An instance of a concrete descendant remains valid.

Translating a Xeto `Query` into executable SPARQL and companion SHACL is
specified separately from the three RDF export profiles. See
[Query Execution and Result Validation](#query-execution-and-result-validation).
