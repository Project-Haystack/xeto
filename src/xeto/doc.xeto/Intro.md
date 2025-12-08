# Overview

Xeto is a data-only programming language to specify and exchange typed data.
It uses a plain text format to create *specs* for data type definitions.
You can think of Xeto as a typed JSON, schema language, or ontology language.
It is designed to provide an expression syntax and type system that programmers
will find familiar to work with data formats like JSON, JSON Schema, and RDF.

Xeto is built around the following key concepts:
- Define data types called *specs*
- Define data as *instances* of specs
- Organize specs and instances into modular libraries called *libs*
- Provide a global cloud repository to share libs, specs, and instances
  at [https://xeto.dev/](https://xeto.dev/)

# Specs

Specs are used to define new scalar or collection data types.  The
primary collection type is called *dict* - a map of key/value
pairs called *slots*.  Dicts are the same thing as JSON objects.
Here is a simple example of dict spec:

```xeto
// User account object
User: Dict <sealed> {
  username: Username  // required string field with constraints
  created: Date       // required date field
  email: Str?         // optional string field
}

Username: Scalar <pattern:" ^[a-zA-Z0-9]+$.">
```

Features of the syntax in the example:
  - Defines a dict type named "User"
  - Defines a scalar type named "Username"
  - Define metadata between angle brackets <>
  - User adds sealed metadata marker to prevent subtyping
  - Username adds pattern metadata for a regex constraint
  - User defines three typed fields or *slots*
  - The "email" field is defined as optional
  - Documentation is added using "//" style comments

# Libs

Xeto *libs* are versioned modules to package up specs and instances.
Libs are designed to enforce a scalable global namespace with well
defined dependencies.  Libs make it easy to build, distribute, and
reuse specs.

Libs are named using a dotted naming convention:
  - 'sys': core type system
  - 'sys.files': Xeto representations of MIME types
  - 'sys.template': templating types
  - 'ph.*': Project Haystack ontology for built environment
  - 'cc.*': community contributions
  - 'com.acme': organization that owns DNS "acme.com"
  - 'foo.*': prefix registered on xeto.dev

The website [https://xeto.dev/](https://xeto.dev/) provides
a centralized, public cloud repository to share Xeto libs.

# JSON

One use case of Xeto is to provide a data validation framework for JSON
data.  Vanilla JSON data only provides a few data primitives (objects,
arrays, strings, numbers, and booleans).  But we can use Xeto's richer
type system to enforce constraints for objects and string scalars.

For example we can validate that the following JSON object against
the 'User' spec from above:

```json
{"username":"bob", "created":"2025-12-08"}
```

Or we could compile the specs above to a JSON schema to use with
existing frameworks.

All of the Xeto tooling allows you to easily export Xeto specs and
data to JSON.  For example the User spec can be compiled to JSON as
follows:

```json
{
  "id": "acme::User",
  "base": "sys::Dict",
  "sealed": "✓",
  "slots": {
    "username": {
      "type": "acme::Username",
      "doc": "required string field"
    },
    "created": {
      "type": "sys::Date",
      "doc": "required date field"
    },
    "email": {
      "type": "sys::Str",
      "maybe": "✓",
      "doc": "optional string field"
    }
  }
}
```

# RDF

Xeto is also designed as a programmer friendly language for ontology definition.
You can export specs and instances to RDF graphs.  For example the spec above
can be exported to an RDF class/SHACL shape:

```
  acme:User
  a rdfs:Class ;
  a sh:NodeShape ;
  rdfs:subClassOf sys:Dict ;
  rdfs:label "User"@en ;
  sh:targetClass acme:User ;
  sh:property [
    sh:path acme:User.username ;

  ...
```

# Project Haystack

Xeto is developed by Project Haystack, a non-profit founded to
define ontologies for the built environment and IoT.  Xeto
is the foundational layer for the Project Haystack 5.0 ontology,
but designed to be universally suitable for any type of data modeling
and validation.  The 'ph.*' suite of libs are managed in the
core Xeto GitHub repo and used for many of the examples.

