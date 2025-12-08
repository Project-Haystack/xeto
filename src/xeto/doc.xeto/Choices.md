# Overview

Choices define exclusive relationship between a set of marker tags.
They are used as "adjectives" in an ontology to add additional constraints
that work in conjunction with the primary inheritance hierarchy. For
example, a sensor measures the `ph::Quantity` of a specific `ph::Phenomenon`.
Both quantity and phenomenon are mutually exclusive.  A given sensor
point cannot report both air temperature and air humidity - it has a
single current value that is one or the other.

# Syntax

Choices leverage spec inheritance to build out a taxonomy of the
choices.  The root of the taxonomy extends `sys::Choice`.  Choice
specs can define only marker type slots.  For example to build a
exclusive set of markers to model color:

```xeto
Color: Choice
Red: Color { red }
Green: Color { green }
Blue: Color { blue }
```

When an entity defines the 'Color' choice it creates a validation rule
that instances must have exactly one of the marker tags: 'red', 'blue',
or 'green'. Here is an example:

```xeto
// color is required
Car: {
  color: Color
}

// instances
Car {red}         // valid
Car {}            // invalid, color is required
Car {red, blue}   // invalid, red and blue are mutually exclusive
```

You can use a [maybe](TypeSystem.md#maybe) choice to define zero
or one choice selection:

```xeto
// color is required
Car: {
  color: Color?
}

// instances
Car {red}          // valid
Car {}             // valid, color is maybe type
Car {red, blue}    // invalid, red and blue are mutually exclusive
```

# MultiChoice

If the choice is not mutually exclusive then add the 'multiChoice'
marker to call site.  This allows instances to make multiple selections
of the choice.  Extending our car example from above:

```xeto
// color is required multi choice
Car: {
  color: Color <multiChoice>
}

// instances
Car {red}          // valid
Car {}             // invalid, at least one color is required
Car {red, blue}    // valid
```
# Scope

Choices are by design open ended. Any lib in the [namespace](Namespaces.md) can
add additional choices via inheritance.  This means that the range of choices
might vary based on what libs are loaded into the namespace.  If a choice
should be a closed set, then add the 'sealed' metadata:

```xeto
Color: Choice <sealed>
```

The sealed marker indicates that only specs within the parent lib
can extend the choice.
