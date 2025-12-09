# Overview

All xeto documentation uses a flavor of markdown we call *xetodoc*.
Xetodoc is formally based on [Commonmark](https://commonmark.org/), with
the following extensions:
  - [Shortcut Links](#shortcut-links): shortcut `[link]` syntax
  - [Tables](#tables): Github table extensions
  - [Videos](#videos): embedded video support for YouTube, Vimeo, and Loom
  - [HTML](#html): nested HTML is explicitly disallowed

# Shortcut Links

Markdown links come in several flavors, but the typical format is `[text](uri)`.
Xetodoc allows this to be shortened to `[uri]`.  Furthermore there is is
built in support for linking by relative or qualified names:

```
[ph.points]        Link to library index
[ph::Thermostat]   Link to spec or instance via qname
```

Test links:
 - [ph.points]: link to library
 - [ph::Thermostat]: link spec via qname

# Tables

Xetodoc allows tables using the Github flavored [table extension](https://github.github.com/gfm/#tables-extension-) format.

Example:

```
| foo | bar |
| --- | --- |
| baz | qux |
```

# Videos

Xetodoc allows embedded videos using the standard image format with
special handling for the "video:" URI scheme.  The following sources
are supported:

1. **YouTube** The format for the YouTube URL is 'video://youtu.be/<id>?si=<si>' or
'video://youtube/<id>?si=<si>'.  The 'id' and 'si' can be found by going
to the YouTube video, clicking the 'Share' button, and inspecting the link text.

2. **Loom** The format for a Loom URL is 'video://loom/<id>?sid=<sid>'.  The 'id'
and 'sid' can be found by going to the Loom video, clicking the "link" icon and
inspecting the link text.

Youtube example:

```
![iframe title](video://youtu.be/fr-K-MVbAa8?si=dnJTcLsVdzfUfE7T)
```

# HTML

Nested HTML is explicitly disallowed in xetodoc and will be ignored
when displaying documentation.

