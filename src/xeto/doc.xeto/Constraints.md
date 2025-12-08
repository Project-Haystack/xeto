# Overview

Xeto includes a built-in set of constraint rules that are used
for data validation.

# Number Constraints

Number specs can be annotated with the following meta to constrain
their value space:

  - 'minVal': inclusive minimum value
  - 'maxVal': inclusive maximum value
  - 'unit': requires the number to have a specific unit
  - 'quantity': requires the number to have a unit of a specific quantity

Examples:

``` xeto
Foo: {
  percent: Number <minVal: 0, maxVal: 100, unit:"%">
  powerVal: Number <quantity:"power">
}

// Slot 'percent': Number 50°C must have unit of '%'
@a: Foo { percent:50°C, powerVal:70kW }

// Slot 'percent': Number 200% > maxVal 100
@b: Foo { percent:200%, powerVal:70kW }

// Slot 'power': Number must be 'power' unit; '°C' has quantity of 'temperature'
@c: Foo { percent:50%, powerVal:70°C }
```

# Str Constraints

Str specs can be annotated with the following meta to constrain
their value space:

  - 'minVal': inclusive minimum length
  - 'maxVal': inclusive maximum length
  - 'nonEmpty': marker tag to require string to have non-whitespace characters
  - 'pattern': regex for required string pattern (example further below)

``` xeto
Foo: {
  name: Str <nonEmpty>
  phone: Str <minVal: 7, maxVal:10>
}

// Slot 'name': String must be non-empty
@a: Foo { name:"", phone:"5551234" }

//  Slot 'phoneNum': String size 3 < minSize 7
@b: Foo { name:"Bob", phone:"555" }

// Slot 'phoneNum': String size 11 > maxSize 10
@c: Foo { name:"Bob", phone:"21255512345" }
```

# List Constraints

List specs can be annotated with the following meta to constrain
their value space:

  - 'minSize': inclusive minimum length
  - 'maxSize': inclusive maximum length
  - 'nonEmpty': marker tag to require at least one item

``` xeto
Foo: {
  listA: List <nonEmpty>
  listB: List <minSize:2, maxSize:3>
}

// Slot 'listA': List must be non-empty
@a: Foo { listA:{}, listB:{"a", "b"}}

// Slot 'listB': List size 1 < minSize 2
@b: Foo { listA:{"x"}, listB:{"a"}}

// Slot 'listB': List size 4 > maxSize 3
@c: Foo { listA:{"x"}, listB:{"a", "b", "c", "d"}}
```

# Scalar Constraints

All scalars must support a string encoding.  The string encoding can
be constrained at compile time using the 'pattern' tag with a regular
expression:

```xeto
SocialSecurityNumber: Scalar <pattern:"\\d{3}-\\d{2}-\\d{4}">

// ok
@a: {ssn: SocialSecurityNumber "123-43-5678"}

// String encoding does not match pattern for 'acme::SocialSecurityNumber'
@b: {ssn: SocialSecurityNumber "123435678"}
```

# Fixed Values

A spec can define a fixed value for a slot via the 'fixed' marker.
A fixed value requires all instances to have that exact value.

``` xeto
Foo: {
  // requires all instances to use the fixed value "%"
  unit: Unit <fixed> "%"
}

// Slot 'unit': Must have fixed value '%'
@a: Foo {unit:"meter"}
```