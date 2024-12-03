# Namespaces

A namespace is a set of [libs](Libs.md) used to build the scope of specs and
instances.  A namespace is defined by a list of lib names and a specific
version of each lib. Namespaces are used to:

  - reflect specs, globals, and lib instances in scope
  - [resolve](#name-resolution) qnames and simple names
  - compute [xmeta](#xmeta)
  - compute taxonomy of [choices](Choices.md)

# Named Definitions

There are three types of named definitions in a lib:
  - **Type Specs**: named via a string starting with an upper case letter
  - **Global Specs**: named via a string starting with a lower case letter
  - **Instances**: named with an Ref identifier

It is a compile time error to declare a spec and instance with the
same name (even though one is a string and one is a Ref). For example
this code will not compile:

```xeto
// invalid code - duplicate top-level names
device: Marker
@device: {}
```

It is valid (and common) to have a type spec and global spec with
the same name that differs only by case:

```xeto
// this is ok
device: Marker
Device: Dict { device }
```

# Lib Namespaces

Libs import external names into their local namespace via their dependencies.
All libs must depend on `lib:sys` (with the exception of sys itself). The
range of all named definitions in the lib itself and its direct dependencies
is called the *lib namespace*.

Only names found in the lib's namespace may be used for spec and instance
definitions.  It is a compile time error to use a name not defined in the
lib namespace.

Code within a lib may use fully qualified names or simple names.  The rules
for resolving names is discussed below in [name resolution](#name-resolution).

# Project Namespaces

Individual projects may pick and choose which libs are used to define their
application specific data. We call this namespace the *project namespace*.
Project namespaces will often be a mix of standardized libs, vendor specific
libs, and project specific libs. It is outside the scope of this specification
to define how libs are included or enabled for project namespaces.

It is recommended that project applications always use fully qualified names
for referencing specs and instances.  But it if an application choses to allow
simple names, then it should follow the standard [name resolution](#name-resolution)
rules discussed below.

# Name Resolution

Names are used to reference a spec or instance.  Names come in four flavors:
  - Fully qualified spec names (qnames)
  - Simple spec name
  - Fully qualified instance identifiers
  - Simple instance identifiers

Examples for the four flavors:

```xeto
ph.points::AirTempSensor  // spec qname
AirTempSensor             // spec simple name
@ph::filetype:json        // qname instance id
@filetype:json            // simple instance id
```

The rules for resolving spec names and instances identifiers are identical;
the only difference is what kind of definition is resolved.

If the name contains a "::" double colon then it is assumed to be a qname
(fully qualified with the global lib name).  Any name missing the
double colon is assumed to be a simple name.

To resolve a qname:
  - resolve the qname lib part to a lib from the namespace dependencies
  - resolve the qname simple name part to a spec/instance within that lib

To resolve a simple name:
  - check all libs in the namespace for an appropriate definition
  - if exactly one definition is found then use it
  - if zero definitions are found it is an unresolved name error
  - if multiple definitions are found it is ambiguous name error

In the case of an ambiguous name error, the code can be fixed by
replacing the simple name with a qname to the correct library.

When [compiling](#lib-namespaces) a lib, its namespace is defined
by the direct dependencies.

# XMeta

Extended meta or *xmeta* is a feature used to add additional
metadata to specs via late binding.  This feature is commonly used
to overlay UI annotations over built-in types.

Adding xmeta to a spec is done by creating [instances](Instances.md)
with a specific naming convention:

```
xmeta-{lib name}-{spec name}
```

For example, to add a color and icon tag to `ph::Room`:

```xeto
@xmeta-ph-Room: {icon:"door", color:"purple"}
```

When resolving the xmeta for `ph::Room` the runtime checks all libs
in the namespace for instances that match the given naming convention
and merges it with the spec's definition metadata.  The runtime will
also check the spec's full inheritance hierarchy for xmeta definitions.

We support a special syntax for using xmeta with an [enum](Enums.md)
to annotate the range items with a single dict using the following
instance naming pattern:

```
xmeta-{lib name}-{spec name}-enum
```

The instance is a nested dict where the keys are the enum range
items and the values are the extended metadata.  Consider an example
to add icons and color to the `ph::CurStatus` type:

```xeto
@xmeta-ph-CurStatus-enum: {
  ok:       {icon::"check,     color:"green"}
  stale:    {icon:"check,      color:"amber"}
  fault:    {icon:"err",       color:"red"}
  down:     {icon:"warn",      color:"orange"}
  disabled: {icon:"ban",       color:"gray"}
  unknown:  {icon:"question"", color:"gray"}
}
```

It is a compile time error to use the xmeta naming pattern for
a spec not found in the lib namespace.



