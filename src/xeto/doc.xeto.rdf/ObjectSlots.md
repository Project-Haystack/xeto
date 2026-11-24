<!--
title:      Object Slots
-->

# Other Core Types

Most dictionary types need no special mapping. A type derived from `Dict` uses
the [class and slot rules](Mappings.md#specs-slots-and-instances) and the
[nested-object rules](#nested-object-slots). For example,
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

ex:Envelope-fields a sh:NodeShape ;
  sh:property [
    sh:path ex:Envelope.failure ;
    # RDF represents the inline Failure dictionary as a blank node.
    sh:nodeKind sh:BlankNode ;
    sh:or ( [ sh:property [ sh:path sys:spec ; sh:maxCount 0 ] ]
            [ sh:class ex:Failure ] ) ;
    # Apply Failure's field checks to the inline dictionary.
    sh:node ex:Failure-fields ;
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

# Unconstrained Object Slots

A slot declared as `Obj` can hold values of different types.

## SHACL Validation

An `Obj` slot constrains presence and single-value cardinality but deliberately
does not constrain the RDF term's datatype, class, or node kind.

```xeto
Envelope : Dict {
  value: Obj
}
```

```turtle
ex:Envelope-fields a sh:NodeShape ;
  sh:property [
    sh:path ex:Envelope.value ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] .
```

# Nested Object Slots

A slot can also specify the type of a nested dictionary.

## SHACL Validation

A slot typed as another dictionary spec uses `sh:node` to apply that spec's
field requirements to the nested value. The property also checks that the value
is a Dict and that its stored spec, when present, fits the slot's type.
This also applies to a slot declared directly as `Dict`. Its value must be
an inline blank node. A Ref to a dictionary record fails that check.

For a bare `Dict` slot, a declared union also fits when every alternative is a
dictionary type. Given `Either: Left | Right` with two Dict alternatives,
`loose: {spec:@example::Either}` fits `loose: Dict`. The fitting predicate
includes `[ sh:class ex:Either ]` alongside `[ sh:class sys:Dict ]` because
the vocabulary links each alternative to the union. The same predicate applies
to each item of `List<of:Dict>`.

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
ex:Person-fields a sh:NodeShape ;
  sh:property [
    sh:path ex:Person.address ;
    sh:nodeKind sh:BlankNode ;
    sh:or ( [ sh:property [ sh:path sys:spec ; sh:maxCount 0 ] ]
            [ sh:class ex:Address ] ) ;
    sh:node ex:Address-fields ;
    sh:minCount 1 ;
    sh:maxCount 1 ;
  ] ;
  sh:property [
    sh:path ex:Person.postalAddress ;
    sh:nodeKind sh:BlankNode ;
    sh:or ( [ sh:property [ sh:path sys:spec ; sh:maxCount 0 ] ]
            [ sh:class ex:PostalAddress ] ) ;
    sh:node ex:PostalAddress-fields ;
    sh:maxCount 1 ;
  ] .
```

Dictionary inheritance may pass through more than one spec. The exporter
follows the complete inheritance chain, but `sh:node` names the slot's declared
type; in this example, `PostalAddress`. If that type is declared by a dependency,
the same rule applies using the dependency library's RDF namespace. A loaded
`addresses::PostalAddress`, for example, becomes
`sh:node addresses:PostalAddress-fields`.

The nested node may itself contain scalar, ref, marker, unit, list, choice, or
further nested values supported by this specification.

For example, let `SpecialAddress: Address { note: Str }`. These two addresses
have the same fields and declare the same type:

```trio
id: @person
spec: @example::Person
address: {id:@home spec:@example::SpecialAddress city:"Here"}
---
id: @home
spec: @example::SpecialAddress
city: "Here"
```

The relevant RDF is:

```turtle
ex:person ex:Person.address [
  a ex:SpecialAddress ; xetoRdf:sourceId ex:home ;
  sys:spec ex:SpecialAddress ;
  ex:Address.city "Here"^^xsd:string
] .
ex:home a xetoRdf:Record, ex:SpecialAddress ;
  ex:Address.city "Here"^^xsd:string .
```

Person's slot asks for Address, so the inline value passes with its `city`.
The separate lookup record at `ex:home` also receives SpecialAddress's field
checks and fails for missing `note`. A list slot with `of:Address` checks each
item against Address in the same way.

## Nested Sugar Types

A nested slot typed as a sugar spec checks both
[type matching and field validation](Mappings.md#type-matching-and-field-validation).
For example:

```xeto
Target: Dict { *hot: Marker, *label: Str }
Hot: Target <sugar> { hot, label: Str "default" }
Envelope: Dict { value: Hot }
```

The following runtime value fits Hot through its `hot` tag and passes Hot's
field checks:

```trio
id: @envelope
spec: @example::Envelope
value: {spec:@example::Target hot label:"ok"}
```

These excerpts show field validation only. The complete constraints also
include the fitting predicate and Hot's marker check:

```turtle
ex:Envelope-fields sh:property [
  sh:path ex:Envelope.value ;
  sh:node ex:Hot-fields ;
  sh:minCount 1 ;
  sh:maxCount 1
] .

ex:Hot-fields sh:property [
  sh:path [
    # Read label through either property; the constraints require
    # one distinct label value.
    sh:alternativePath ( ex:Target.label ex:Hot.label )
  ] ;
  sh:datatype xsd:string ;
  sh:minCount 1 ;
  sh:maxCount 1
] .
```

Hot's shape checks the fields using their applicable RDF properties. The
`label` constraint accepts either `ex:Target.label` or `ex:Hot.label`; the
property list is abbreviated to these types.

Removing `label` makes the nested `value` invalid: Hot requires a string field
named `label`, even though the default does not participate in sugar matching.
A non-Marker `hot` value can satisfy the presence test but fails field validation.
If the slot were instead `value: Ref<of:Hot>`, a referenced Target record with
`hot` but no `label` would still satisfy the reference constraint. Reference
fitting does not apply Hot's full field validation; see
[Ref Slots](References.md#shacl-validation).

Without a `spec` field, a nested dictionary can proceed directly to field
validation. The missing type declaration alone does not cause failure.

## Instance Data

Each nested dictionary becomes a blank node. Its fields describe the value
stored in the enclosing slot. For example, when an `address: Address` value
omits `spec`, the exporter uses `Address` for the RDF class and field
properties. Only a stored `spec` field produces a `sys:spec` triple.

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
ex:person1 a xetoRdf:Record, ex:Person ;
  ex:Person.address [
    a ex:Address ;
    ex:Address.city "Richmond"^^xsd:string
  ] ;
  ex:Person.postalAddress [
    a ex:PostalAddress ;
    ex:Address.city "Richmond"^^xsd:string
  ] .
```

When the dictionary contains a Ref-valued `id`, the blank node carries that
IRI as `xetoRdf:sourceId`. An absent `id` produces no triple. A malformed
literal ID retains its value. For example:

```trio
address: {id:42 city:"Here"}
```

```turtle
ex:person1 ex:Person.address [
  a ex:Address ; xetoRdf:sourceId "4.2E1"^^xsd:double ;
  ex:Address.city "Here"^^xsd:string
] .
```

If Address declares `id: Ref?`, its ID constraint is:

```turtle
ex:Address-fields sh:or (
  [ sh:nodeKind sh:IRI ]
  [ sh:nodeKind sh:BlankNode ;
    sh:property [ sh:path xetoRdf:sourceId ; sh:nodeKind sh:IRI ; sh:maxCount 1 ] ]
) .
```

The inline address may omit its ID. The supplied numeric literal fails
`sh:nodeKind sh:IRI`. For a required `id: Ref`, the property also has
`sh:minCount 1`. A non-Ref ID without a literal mapping uses a blank node
so it also fails the IRI check.

For a compiled library, `Lib.instances` supplies the
lookup records. A named declaration inside another record appears in that
collection, so the exporter also describes the lookup record once at its IRI.

```xeto
@person1: Person {
  address @address1: Address {
    city: "Richmond"
  }
}
```

```turtle
ex:person1 ex:Person.address [
  a ex:Address ;
  xetoRdf:sourceId ex:address1 ;
  ex:Address.city "Richmond"^^xsd:string
] .

ex:address1 a xetoRdf:Record, ex:Address ;
  ex:Address.city "Richmond"^^xsd:string .
```

Runtime input can supply different values under the same id. For example,
given `Person: Dict { address: Address, targetRef: Ref<of:Address> }`:

```trio
id: @start
spec: @example::Person
address: {id:@home spec:@example::Address city:"Here"}
targetRef: @home
---
id: @home
spec: @example::Address
city: "There"
```

Reading the inline address yields `"Here"`. Following `targetRef` yields the
lookup record with city `"There"`. Removing the second record leaves the inline
address intact. The Query path `via:"targetRef"` then returns no record, and
field validation reports a dangling Ref.
