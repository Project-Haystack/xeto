# Overview

Libs and specs can define arbitrary *meta* or metadata.  Many of xeto's
built-in features leverage meta to annotate specs.  For example, the `sealed`
meta is used by the compiler to indicate that a spec cannot be subtyped.
Both lib and spec meta is a normal dict, but uses special rules for
static  typing.  Spec meta tags are defined as slots on the `sys::Spec`
type and lib meta tags on `sys::Lib`.  You can use [mixins](Mixins.md)
to add new spec/lib meta tags.  Note all Spec/Lib meta tags must be defined
as nullable (since they always optional).

# Syntax

New meta tags are defined as mixin slots:

```xeto
+Spec {
  // new spec meta tag
  specMetaTag: Type?
}

+Lib {
  // new lib meta tag
  libMetaTag: Type?
}
```

