# Foundations

## Mapping Context

### Namespace

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

### Complete Spec Definition

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

The [Mixins](TypeComposition.md#mixins) section describes how the contributed `orgRef` slot keeps
its `app.people` identity in RDF without creating a separate `app.people::Person`
class.

### Concrete Library Versions

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

## RDF Graphs

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
add `sh:closed true` unless a construct's mapping rule explicitly requires it.

## Namespaces and IRIs

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
this specification:

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
for the optional ontology-packaging terms described in
[Metadata and External Vocabularies](Metadata.md#metadata-and-external-vocabularies).
The mapping itself does not require OWL inference.

### Turtle-safe names and values

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

