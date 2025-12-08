# Overview

All xeto documentation uses a flavor of markdown we call *xetodoc*.
Xetodoc is formally based on [Commonmark](https://commonmark.org/), with
the following extensions:
  - [Doc Links](#doc-links): the backtick is used for doc hyperlinks
  - [Inline Code](#inline-code): the single quote is used for inline code
  - [Tables](#tables): Github table extensions
  - [Videos](#videos): embedded video support for YouTube, Vimeo, and Loom
  - [HTML](#html): nested HTML is explicitly disallowed

# Doc Links

Standard markdown uses the backtick to denote inline code text.
In xetodoc these are treated as hyperlinks to the reference
documentation.  The following syntax is supported:

```
`ph.points`             Link to library index
`ph.points::FanPoint`   Link to spec, global, or instance via qname
```

# Inline Code

Since we use the backtick for code links, we also support creating inline
code using the single quote:

```
This is a `link`, but this is 'inline text'
```

# Tables

Xetodoc allows tables using the Github flavored [table extension](https://github.github.com/gfm/#tables-extension-) format.

Example:

```
| foo | bar |
| --- | --- |
| baz | bim |
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

