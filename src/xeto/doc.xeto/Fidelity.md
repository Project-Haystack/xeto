# Overview

Xeto provides an extensible, opened ended type system.  Translating Xeto data
to other data formats with less flexible type systems results in type erasure.
We the use term *fidelity* to precisely describe how type erasure occurs.

Three levels of fidelity are defined:

- **Full Fidelity**: all types are completely modeled with no loss of fidelity
- **Haystack Fidelity**: only the core Haystack types are used and
  all scalars encode as simple strings
- **JSON Fidelity**: most scalars are encoded as just strings

# Full Fidelity

Full fidelity requires that there is zero type erasure.  All types
(including scalars) must fully describe their type.  Implementations
must take special care how scalars are treated.  Common low-level types
such as 'Bool', 'Int', and 'Str' are sealed to more easily map to the
native types of the implementation programming language.

The only data serialization format that supports full fidelity is the
Xeto format itself.

The Haxall reference implementation uses the following strategy:
1. Map all system level specs to built-in Fantom classes
2. Allow users to customize mapping specs to Fantom classes via a factory framework
3. If there is no Fantom class mapping for a scalar we use a generic Scalar
   class that tracks the type qname and string encoding

This design allows us to track all objects in memory back to their Xeto specs
with no loss of fidelity. This strategy is non-normative, other implementations
are free to tackle the problem however they they see fit.

# Haystack Fidelity

The Project Haystack type system provides a fixed set of types similar to
that found in XML Schema.  The Haystack kinds map one-to-one to Xeto
specs as follows:

```
  Xeto              Haystack
  --------          -----
  sys::Marker       Marker
  sys::NA           NA
  sys::None         Remove
  sys::Bool         Bool
  sys::Number       Number
  sys::Int          Number
  sys::Float        Number
  sys::Duration     Number
  sys::Str          Str
  sys::Ref          Ref
  sys::Uri          Uri
  sys::Date         Date
  sys::Time         Time
  sys::DateTime     DateTime
  sys::List         List
  sys::Dict         Dict
  sys::Grid         Grid
  ph::Coord         Coord
  ph::Symbol        Symbol
```

Any scalar not mapped above is encoded in Haystack as a simple string.
All dicts must declare a 'spec' tag to declare their Xeto spec.
Lists are untyped in Haystack.  We do not currently define a mapping
for XStr - they should map to a specific scalar spec.

Haystack fidelity is used when exchanging or validating data encoded
using Haystack formats such as Zinc or Hayson.

# JSON Fidelity

JSON is widely used as a data interchange format.  JSON supports a very
limited type system, which requires significant type erasure.  We opt for
a clean, simple JSON mapping where most scalars are mapped to a string (versus
mapping scalars to JSON objects).  The following mapping is used:

```
  Xeto           JSON
  ----           ----
  sys::Bool      boolean
  sys::Int       number
  sys::Float     number
  sys::Number    number  (if no unit)
  sys::Number    string (if there is a unit)
  sys::Scalar    string  // all other scalars encode as string
  sys::List      array
  sys::Dict      object
  ```

All dicts should include a 'spec' tag with the type qname.
