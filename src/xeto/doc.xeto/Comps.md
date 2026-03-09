# Overview

The [sys.comp::index] lib provides a standard ontology for modeling heterogeneous
component oriented applications.  It does not define any implementation behavior,
rather it only provides a light weight set of standardized types for modeling
common concepts found in control-flow or workflow frameworks.

  - [Comp](#comp): base type for components or function blocks
  - [CompLayout](#complayout): standardized layout on 2D wiresheet
  - [CompFunc](#compfunc): component object-oriented methods
  - [Links](#links): standardized model for wiring data flow between components

This core `sys.comp` library defines nothing related to implementation.  It
is purely for modeling component definitions and a graph of instances.

# Comp

The [sys.comp::Comp] type provides a basic interface for all components
that wish to use the standard Xeto model.

- if your model organizes components into a tree then use `compName` and `parentRef`
  to define the tree structure
- if your model provides 2D layout, then use `compLayout` using a logical coordinate system
- if your model supports data flow links, then use `links` to model them
  via a standardized graph design

Comps are defined in Xeto as normal types and their slots specify the
fields and methods:

```xeto
MyAddBlock : Comp {
  in1: Number <transient>
  in2: Number <transient>
  out: Number <transient>
  execute: Func
}
```

# CompLayout

The [sys.comp::CompLayout] scalar type encodes the coordinates and width of
component/block into a logical 2D coordinate system.  The height is assumed
to be defined by the component's slots.

```xeto
// this specifies that top, left corner of component is at x=5, y=10
// and width is 4
compLayout: "5, 10, 4"
```

# CompFunc

If your component model has the concept of object oriented methods, then you
can use the [sys.comp::CompFunc] type.  Component functions should define
exactly one parameter, this makes them suitable for wiring sensibly into
data flow applications via links.

The `CompFunc` type is designed to support component method functions that
can be defined statically as normal function specs or dynamically in instance
models.  Static definitions use the standard Func pattern where parameter
and return types are defined as slots.  Instance definitions must reference
a predefined Func type via the `funcType` tag. Or if `funcType` is omitted
we default to a `arg:Obj? -> Obj?` function type signature:

```xeto
// static slot definition
MyFuncBlock: {
  doIt: Func { arg: Str, returns: Str }
}

// reusable function signature
StrToStrFunc: Func { arg: Str, returns: Str }

// dynamic instance definition
@xyz: MyFuncBlock {
  doIt: CompFunc { funcType: @acme::StrToStrFunc }
}
```

# Links

The [sys.comp::Link] type defines a standardize model for data flow links
between two component slots.  A link requires four identifiers to fully
identify the relationship:

```
fromComp.fromSlot => toComp.toSlot
```

The way the model captures these four identifiers is as follows:
- We store a `Links` dict on the *toComp* in the slot named `links`
- We store a `Link` dict inside `Links` dict using the *toSlot* name
- The `Link` dict stores the *fromComp's* id using `fromRef`
- The `Link` dict stores the *fromSlot* name using `fromName`

Here is example:

```xeto
// fromComp that outputs data in slot named "out"
@a: DataProducer {
}

// toComp that links out to its own input slot named "in"
@b: DataConsumer {
  links: {
    in: Link { fromRef:@a, fromSlot:"out" }
  }
}
```

