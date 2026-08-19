# Overview

Grids are the tabular data structure defined by [sys::Grid].  A grid
is Xeto's dataframe: the workhorse type for query results, history
data, imports and exports, and any data naturally shaped as rows and
columns.  Functions in the [HTTP API](HttpApi.md) commonly accept and
return grids.

A grid is composed of:

- Grid level metadata
- Columns, each with a name, optional type, and metadata
- Rows, each a dict whose tags map to the column names

# Shape

Xeto defines a formal shape for grids so they can be exchanged as
instance data and declared as constants inside libs.  It is the same
dict shape used by [Jeto](Jeto.md#grids) rendered in Xeto syntax.  A
grid is instance data, so the shape never uses the `<>` meta syntax -
it is nested curly braces only.

A grid is always a value: it stands alone in a data file or nests
as a slot value within a named instance.  A grid itself cannot be
a named instance with an id, the same rule as List.

A grid is a dict with the following slots:

- The type declares the grid spec: `Grid` or a subtype
- An optional `of` slot declares the default spec for every row
- An optional `meta` slot holds the grid level metadata
- A `cols` slot lists the name, type, and metadata of each column
- A `rows` slot lists each row as a dict

For example:

```xeto
Grid {
  meta: {foo: "quux"}
  cols: {
    {name: "a"},
    {name: "b", meta: {dis: "B"}}
  }
  rows: {
    {a: 0, b: "x"},
    {a: 1, b: "y"}
  }
}
```

# Cell Types

Grids may declare types for their cells in three places:

- A column may declare an `of` slot beside its `name`, which gives
  a type to that column's cells
- A grid may declare an `of` slot, which gives a default spec to
  every row
- A row may declare its own type via `spec` tag, which overrides the
  default row spec for that row.  The rows of a grid need not all be the same type.

The `of` slots sit beside the thing they describe rather than inside
its `meta` because they are structural rather than domain tags; `meta`
holds only domain tags at both the grid and column level.

The precedence for typing a cell: a cell or row that declares its own
type keeps it, then the column `of`, then the row spec member, where
the row spec is the row's own type else the grid `of`.  A column `of`
that contradicts a typed row's declared member is an error.  The row
type governs how its cells decode; the rows of the resulting grid are
columnar, so the type itself is not carried as a cell.

For example:

```xeto
Grid {
  cols: {
    {name: "ts", of: DateTime},
    {name: "val", of: Number}
  }
  rows: {
    {ts: "2026-01-01T00:00:00Z", val: 72°F},
    {ts: "2026-01-01T00:15:00Z", val: 73°F}
  }
}
```

The `of` values are spec names, not `@` refs: an `@` ref resolves in
the instance space.  A simple name resolves against the namespace or
use the qualified name such as `sys::DateTime`.

# Jeto

The shape corresponds to [Jeto](Jeto.md#grids) as follows:

| Jeto                  | Xeto                     |
| ----                  | ----                     |
| grid `spec` entry     | grid type                |
| grid `of` entry       | `of` slot                |
| row `spec` property   | row type                 |
| `meta`, `cols`, `rows`| same slots               |

