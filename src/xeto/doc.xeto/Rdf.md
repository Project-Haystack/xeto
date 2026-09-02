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
  sys:hasMarker ex:Site.site .

ex:equip1 a sys:Entity, ex:Equip ;
  ex:Equip.dis "AHU-1"^^xsd:string ;
  ex:Equip.siteRef ex:site1 .
```

The subject IRI carries the Xeto id. No additional id predicate is required by
this profile.

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

## Scalar Slots

### SHACL Validation

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

A custom spec derived from `Scalar` maps to `xsd:string`; its scalar
constraints, such as `pattern`, are expressed in SHACL.

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

String `minSize`, `maxSize`, `nonEmpty`, and scalar `pattern` metadata map to
`sh:minLength`, `sh:maxLength`, and `sh:pattern`. `nonEmpty` uses the pattern
`\\S`, which requires at least one non-whitespace character.

```xeto
Code : Scalar <pattern:"[A-Z]{2}">

Asset : Dict {
  code: Code
  dis: Str <nonEmpty, minSize:2, maxSize:40>
}
```

```turtle
ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.code ;
    sh:datatype xsd:string ;
    sh:pattern "[A-Z]{2}" ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
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

A custom scalar may refine another custom scalar. Every pattern in that
inheritance chain remains part of the value contract.

```xeto
UpperCode : Scalar <pattern:"[A-Z]+">
FourCharacterCode : UpperCode <pattern:"[A-Z0-9]{4}">

Asset : Dict {
  code: FourCharacterCode
}
```

```turtle
ex:Asset a sh:NodeShape ;
  sh:targetClass ex:Asset ;
  sh:property [
    sh:path ex:Asset.code ;
    sh:datatype xsd:string ;
    sh:pattern "[A-Z]+" ;
    sh:pattern "[A-Z0-9]{4}" ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

The two patterns are combined: a value must satisfy both the rule inherited
from `UpperCode` and the refinement declared by `FourCharacterCode`. The same
rule applies when a scalar base comes from a loaded dependency. If the same
pattern appears at more than one level, the exporter emits it once; a pattern
declared on the slot is combined with the inherited type patterns.

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
| `None` | RDF absence is represented by no statement; the `None` value needs a separate mapping. |
| `NA` | The not-available sentinel needs an RDF representation distinct from an ordinary string. |
| `Grid` | A grid mapping must define rows, columns, cells, metadata, and ordering. |
| `Collection` | The abstract collection type does not identify a concrete RDF value representation. |
| `Func` | Function parameters, return values, and execution are outside the current data profiles. |
| `Interface`, `Funcs`, and `File` | API contracts, callable operations, and file handles are outside the current data profiles. |
| `BuildVar` | A build variable needs an explicit rule for resolved and unresolved build-time values. |

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
RDF property. Enum values are strings rather than RDF individuals, so the
vocabulary graph does not create a resource for each entry.

```turtle
ex:Suit a sys:Class, rdfs:Class ;
  rdfs:subClassOf sys:Enum ;
  rdfs:label "Suit"@en .

ex:Card.suit a rdf:Property ;
  rdfs:label "suit"@en .
```

### SHACL Validation

Enum slots are string-valued and use `sh:in` for the allowed values. For the
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
  ex:Equip.siteRef ex:site1 .

ex:equip3 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 .

ex:equip1 a sys:Entity, ex:Equip ;
  ex:Equip.siteRef ex:site1 ;
  ex:Equip.related ex:equip2, ex:equip3 .
```

The single `siteRef` value produces one statement. The two `related` values
produce two statements with the same subject and property. Referenced Xeto
entities are exported with their own `sys:Entity` and concrete spec types so
typed and untyped reference shapes can validate them.

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

## Queries

### SHACL Validation

A Xeto `Query` is computed navigation, not a stored required field. Query slots
therefore have no `sh:minCount` or `sh:maxCount`. Their `of` type becomes
`sh:class`, and `via` or `inverse` metadata becomes a SHACL property path.

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
    sh:class ex:Point ;
  ] ;
  sh:property [
    sh:path [
      sh:oneOrMorePath [ sh:inversePath ex:Point.equipRef ]
    ] ;
    sh:class ex:Point ;
  ] .

ex:Point a sh:NodeShape ;
  sh:targetClass ex:Point ;
  sh:property [
    sh:path [ sh:oneOrMorePath ex:Point.equipRef ] ;
    sh:class ex:Equip ;
  ] .

ex:Node a sh:NodeShape ;
  sh:targetClass ex:Node ;
  sh:property [
    sh:path [ sh:zeroOrMorePath ex:Node.next ] ;
    sh:class ex:Node ;
  ] .
```

A Query does not require a particular number of matches. Its path may find no
values, one value, or many values. Every value it finds must fit the Query's
`of` type. A `+` path follows the named slot one or more times. A `*` path also
includes the record being checked before following the slot, so that record
must also fit the `of` type. Cycles are allowed and do not cause the traversal
to continue forever.

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
but still has no cardinality constraint. Required members declared inside a
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
    sh:class ex:Point ;
  ] ;
  sh:property [
    sh:path [ sh:inversePath ex:Point.equipRef ] ;
    sh:class ex:Point ;
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
    sh:class base:Point
  ] ;
  sh:property [
    sh:path base:Equip.basePointRef ;
    sh:class base:Point
  ] ;
  sh:property [
    sh:path [ sh:inversePath app:Point.equipRef ] ;
    sh:class base:Point
  ] .
```

For `pointsVia`, the unqualified `pointRef` resolves in the contributing
`app.equips` library, producing `app:Equip.pointRef`. For `pointsViaBase`, the
full name explicitly selects `base:Equip.basePointRef`. This determines which
RDF relationships SHACL follows; the two paths are not interchangeable.

For `pointsInverse`, `Point.equips` first resolves to the query contributed by
`app.equips`. That query traverses `app:Point.equipRef`, so reversing it
produces `[ sh:inversePath app:Point.equipRef ]`.

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
specs. It does not necessarily require every descendant instance to carry that
slot. In this example, `maybe` makes `height` optional, but any descendant that
declares or supplies `height` must preserve the global contract:

```xeto
Person : Dict {
  *height: Number <minVal:0, maxVal:300, maybe>
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

- Required members inside a `Query` body. Their validation requires graph-mode
  query traversal plus qualified SHACL value constraints.
- `List` item types for enums and choices, pending Xeto/Haystack instance
  fidelity fixes.
- `List` item types for units and refs, and nested `List` values, pending a
  complete instance and validation mapping.
- Built-in scalar types not listed in the scalar datatype table, including
  `None`, `NA`, `Duration`, `Version`, `Buf`, `Span`, `Filter`, and
  `BuildVar`, pending explicit RDF datatype and lexical-form mappings.
- `UnitQuantity` values. Quantity metadata on `Unit` and `Number` slots remains
  supported as defined in [Units and Quantities](#units-and-quantities).
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

SPARQL generation for executing a Xeto `Query` as an application query is also
outside these three profiles. The SHACL query-path mapping above preserves
validation traversal; a future query API can use equivalent SPARQL property
paths without requiring stored reverse triples.
