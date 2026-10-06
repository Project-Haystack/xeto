<!--
title:      Types
author:     Brian Frank
copyright:  Copyright (c) 2026, Project-Haystack
license:    Licensed under the Academic Free License version 3.0
-->

# Overview
The sys library defines the core types used to represent all data.
Every value is an instance of [sys::Obj], which branches into two
fundamental kinds of types:
  - [sys::Scalar]: atomic values with a canonical string encoding
  - [sys::Collection]: compound values: dicts, lists, and grids

There are three singleton scalars:
  - [Marker](#marker): label for a "type" or "is-a" relationship
  - [None](#none): absence of a value or a removal operation
  - [NA](#na): not available, missing, or invalid data

The scalar atomic types:
  - [Bool](#bool): boolean "true" or "false"
  - [Number](#number): integer or floating point number with an
    optional unit of measurement
  - [Str](#str): string of Unicode characters
  - [Uri](#uri): Universal Resource Identifier
  - [Ref](#ref): reference used to identify an entity instance
  - [Date](#date): an ISO 8601 date as year, month, day
  - [Time](#time): an ISO 8601 time as hour, minute, seconds
  - [DateTime](#datetime): an ISO 8601 timestamp paired with a timezone
  - [Span](#span): a time range between two DateTimes
  - [Version](#version): decimal digits separated by dots
  - [Buf](#buf): binary data in base64url encoding

And the collection types:
  - [Dict](#dict): associative array of name/value pairs
  - [List](#list): ordered list of zero or more values
  - [Grid](#grid): two dimensional table of columns and rows

This chapter catalogs each type and its literal syntax in Xeto source.
How these types compose into new specs is covered by the
[Specs](Specs.md) and [Type System](TypeSystem.md) chapters; how they
map to Haystack and JSON is covered by the [Fidelity](Fidelity.md)
chapter.

# Literals
Scalar values in instance data are written as one of the following
token formats - see [Grammar](Grammar.md#scalars) for details:
  - quoted strings such as `"hi"` (also triple quoted strings and heredocs)
  - number literals that start with a digit such as `45°F` or `2023-03-04`
  - refs that start with "@" such as `@ahu-3`

The scalar's type is inferred from its slot spec, or may be declared
explicitly by prefixing the literal with a type name:

```xeto
@example: Dict {
  a: "string"             // Str by default
  b: 45°F                 // Number literal
  c: 2023-03-04           // Date literal
  d: Time "14:30:00"      // explicitly typed
  e: Version "5.0.3"      // explicitly typed
}
```

# Marker
[sys::Marker] is a singleton used to create "label" tags that express
typing information - the presence of the tag itself is what carries
meaning.  In Xeto source a marker tag is just a name without a value:

```xeto
Employee: Dict {
  dis: Str
  fullTime: Marker?
  remote: Marker?
}

@employee-27: Employee {
  dis: "Alice Smith"
  fullTime, remote
}
```

The canonical string encoding of the marker singleton is "✓".

# None
[sys::None] is a singleton that represents the absence of a value.
Its canonical string encoding is "∅".  It maps to the Remove value
in Haystack, which indicates removal of a tag in update operations.

# NA
[sys::NA] is a singleton for *not available*.  It is a placeholder for
missing or invalid data, most often used in historized data to indicate
that a timestamp sample is in error.  Its canonical string encoding
is "na".

# Bool
[sys::Bool] is the truth data type with the two values `true`
and `false`.

# Number
[sys::Number] is an integer or floating point value with an optional
unit of measurement.  Implementations should represent a number as a
64-bit IEEE 754 floating point and provide 52 bits of lossless integer
representation.

Units must be a symbol defined by the standard
[unit database](Units.md), which is modeled as the [sys::Unit] enum:

```xeto
45             // unitless integer
-23.45         // unitless floating point
5.4e-7         // unitless exponent format
45°F           // integer with unit
-23.45m²       // floating point with unit
10_000         // underbar is allowed as a separator
5.4E+8kW       // exponent format with unit
```

The three special values NaN, INF, and -INF are written as quoted
strings since number literals must start with a digit:

```xeto
Number "NaN"   // not a number - unit is invalid
Number "INF"   // positive infinity
Number "-INF"  // negative infinity
```

The following subtypes restrict the number value space:
  - [sys::Int]: unitless integer
  - [sys::Float]: unitless floating point
  - [sys::Duration]: number with a unit of time

Number slots may be further constrained with meta such as 'minVal',
'maxVal', 'quantity', and 'unit' - see
[Constraints](Constraints.md#number-constraints).

# Str
[sys::Str] is a sequence of zero or more Unicode characters.  All text
formats must be encoded using UTF-8 unless explicitly specified
otherwise.  Strings are written as double quoted strings with C style
backslash escapes, triple quoted strings, or heredocs - see
[Grammar](Grammar.md#scalars).

# Uri
[sys::Uri] is the data type used to represent Universal Resource
Identifiers according to
[RFC 3986](http://tools.ietf.org/html/rfc3986).  Uris are written
as quoted strings with their type inferred or declared explicitly:

```xeto
@acme: Org {
  website: Uri "https://acme.com/"
}
```

# Ref
[sys::Ref] is the data type for instance data identifiers.  All
entities are identified via the [sys::Entity.id] tag and a unique ref
value.  Relationships cross-reference entities with ref tags.  Refs are
written with a leading "@":

```xeto
@room-204: Room {
  spaceRef: @floor-2
}
```

Refs must adhere to the following limited set of ASCII characters:
  - ASCII lower case letter a-z
  - ASCII upper case letter A-Z
  - ASCII digit 0-9
  - Underbar "_"
  - Colon ":"
  - Dash "-"
  - Period "."
  - Tilde "~"

Double colon "::" is reserved for the qnames of specs and instances
defined within a lib.  Client software must treat refs as opaque
identifiers and never assume they are globally unique.

The `of` meta parameterizes a ref slot with its target type.  Ref may
be subtyped to classify relationship semantics such as
[sys.refs::ContainedByRef]; subtypes are nominal metadata on the spec
only, never value identity - a ref value always reflects as [sys::Ref].

The [sys::MultiRef] type models a value that may be either a single
ref or a list of refs.

# Date
[sys::Date] is an ISO 8601 calendar date encoded as YYYY-MM-DD
such as `2023-03-04`.

# Time
[sys::Time] is an ISO 8601 time of day encoded as hh:mm:ss.sss
such as `14:30:00`.

# DateTime
[sys::DateTime] is an ISO 8601 timestamp paired with a timezone name.
All timestamps must include their actual timezone versus just a UTC
offset; the one exception is UTC itself, where the name may be omitted
if the offset is "Z".  Timezone names are standardized by the
[timezone database](TimeZones.md), which is modeled as the
[sys::TimeZone] enum:

```xeto
DateTime "2026-08-19T16:50:23-04:00 New_York"   // eastern daylight time
DateTime "2026-08-19T20:50:23Z UTC"             // same time in UTC
DateTime "2026-08-19T20:50:23Z"                 // may omit timezone if offset is Z
DateTime "2026-08-19T14:50:23.507-06:00 Denver" // fractional seconds
```

# Span
[sys::Span] models a time range between two DateTimes.  It supports
the following string encodings:
  - relative [sys::SpanMode] name such as "yesterday" or "thisMonth"
  - absolute single date: `YYYY-MM-DD`
  - absolute date span: `YYYY-MM-DD,YYYY-MM-DD`
  - absolute date time span: `YYYY-MM-DDThh:mm:ss.FFF zzzz,YYYY-MM-DDThh:mm:ss.FFF zzzz`

In the case of two DateTimes, the timezones must match.  In the other
cases the timezone to use is context sensitive based on the application.

# Version
[sys::Version] is a version string formatted as decimal digits
separated by a dot such as `5.0.3`.

# Buf
[sys::Buf] is a blob of raw binary data in base64url encoding as
specified by RFC 4648.  This encoding does not use padding and uses
minus "-" and underbar "_" as the 62nd and 63rd char respectively.

# Dict
[sys::Dict] is an associative array of name/value pairs that we
call *tags*.  Dicts are the universal compound type: all
[instances](Instances.md) are dicts, and dict specs define their
shape via [slots](Specs.md#dicts).

# List
[sys::List] is an ordered list of zero or more values.  The item type
may be parameterized via the 'of' meta.  Lists are written with curly
braces in instance data - see [Specs](Specs.md#lists):

```xeto
@foo-1: Foo {
  numbers: List <of:Number> { 2, 3, 4 }
}
```

# Grid
[sys::Grid] is a two dimensional table of columns and rows, where
each row is a dict.  Grids are the workhorse type for query results,
history data, and data exchange.  See the [Grids](Grids.md) chapter.

