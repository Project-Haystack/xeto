# Overview

The Xeto HTTP API is the network protocol for calling functions on a
server over HTTP.  Any function published with the `op` marker is an
operation clients may invoke by URI.  The core operations are declared by
[sys.api::index]; servers may add additional libs such as [ph.api::index].

The API is version 5 of the Haystack REST API.  It shares its transport
with [version 4](ph.doc::HttpApi) - same URIs, same authentication, same
format table - and differs in four behaviors covered in [Versions](#versions)
below.  Its primary encoding is [Jeto](Jeto.md): plain JSON typed
by the xeto function signature.

Because every operation declares typed parameters and a typed return,
the whole API exports as a [Swagger](https://swagger.io/) document - see
[JSON Schema](#json-schema).  Standard Swagger tooling can browse the
API interactively and generate typed client stubs, which the version 4
grid model could never describe.

Operations are invoked as path names under a base URI the server
defines.  The base must be assumed to end with a trailing slash even if
it is not always expressed that way:

    {base}/{op}

The op name is the function's simple name, or its qname such as
`sys.api::about` when the simple name is ambiguous in the namespace.

The base URI is the server's choice: Haxall serves each project at
`/api/{proj}/`, with the reserved name `sys` addressing the system
runtime itself, while other servers use bases such as `/haystack/`.
Examples in this chapter use the Haxall form.

## Versions

The client selects the protocol version with the `Xeto-Version` request
header, or the `xeto-version` query parameter for easy testing:

    Xeto-Version: 5

The default when neither is given is version 4, so existing Haystack
clients are unaffected.  An unsupported token answers 400
[sys.api::UnsupportedVersionErr] naming the supported versions; note
version resolution happens after authentication, so an unauthenticated
request answers its auth challenge before any version error.  Version 5
responses echo the version back in the same header, and every
[error response](#errors) reports the server's current version.

Both versions accept and serve the same formats.  They differ in four
behaviors:

| Behavior                    | Version 4                  | Version 5                         |
| ----                        | ----                       | ----                              |
| Operation model             | every exchange is a grid   | typed parameters and return       |
| Bare `application/json`     | [Hayson](ph.doc::Hayson)   | [Jeto](Jeto.md)                   |
| Errors from an op           | 200 plus error grid        | HTTP status code and JSON ApiErr  |
| Default with no Accept      | `text/zinc`                | `application/json`                |

## Filetypes

Both versions share one table of formats.  Each format has a
programmatic name used for the `xeto-filetype` query parameter, and a
mime type used with the `Content-Type` and `Accept` headers:

| Name    | Mime Type                        | Spec                     | Doc                        |
| ----    | ----                             | ----                     | ----                       |
| zinc    | `text/zinc`                      | [sys.files::ZincFile]    | [Zinc](ph.doc::Zinc)       |
| trio    | `text/trio`                      | [sys.files::TrioFile]    | [Trio](ph.doc::Trio)       |
| hayson  | `application/vnd.haystack+json`  | [sys.files::HaysonFile]  | [Hayson](ph.doc::Hayson)   |
| csv     | `text/csv`                       | [sys.files::CsvFile]     | [Csv](ph.doc::Csv)         |
| jeto    | `text/jeto`                      | [sys.files::JetoFile]    | [Jeto](Jeto.md)            |
| xeto    | `text/xeto`                      | [sys.files::XetoFile]    | [Instances](Instances.md)  |

Bare `application/json` resolves per version as covered in [Versions](#versions).
The hayson mime also resolves with an explicit `;version=4` param. The deprecated
version 3 JSON uses `application/vnd.haystack+json;version=3` if supported.
The [sys.api::filetypes()] function is the authoritative catalog for an endpoint,
listing every registered format with its mime type and read/write support.

## Operations

An op is a function marked `op` in its declaring lib.  A function also
marked `noSideEffects` may be invoked with GET; every op may be invoked
with POST.  A GET of an op with side effects answers 405.

The [sys.api::ops()] function lists the ops available in the namespace with
their signatures.  It is a discovery and debugging summary, not a schema; the
[Swagger export](#json-schema) serves machine consumption.  Also see [sys.api::libs()]
operation to list the libraries composing the namespace by name and version.

## GET Requests

Query parameters supply the arguments by name.  Each value is a bare
scalar rather than JSON text so URLs stay readable, and is decoded against
its declared parameter spec:

    GET /api/demo/readById?id=xyz-123&checked=false

A value beginning with `[` or `{` is taken as JSON so lists and dicts can
still be expressed.  A parameter with no value in the request falls back
to its declared default; a missing required parameter answers 400.

Parameter names prefixed `xeto-` are reserved for the request itself and
never map to an argument: `xeto-version` selects the protocol version and
`xeto-filetype` the response [filetype](#filetypes).  The prefix can
never collide with a real parameter because a xeto name cannot contain a
dash.

## POST Requests

The `Content-Type` header selects how the body is decoded; a request
without one answers 415.  With `application/json`, `text/jeto`, or
`text/xeto` the body is one object whose members are the named
arguments, each decoded against its own parameter spec under the
[Jeto](Jeto.md) rules:

    POST /api/demo/readById
    Content-Type: application/json

    {"id":"xyz-123", "checked":false}

An op whose parameters all have defaults may be posted with no body at
all.

A Haystack grid format (zinc, trio, hayson, csv) decodes the body as a
grid: the first row's cells supply the arguments by name, which is also
how a version 4 client posts to a modeled op.  A version 4 request
carries the request grid in every format.

The batch and tabular ops such as `hisWrite` and the watches keep the
grid itself as their contract: they declare the `opGrid` marker and a
single `req: Grid` parameter.  A named args client passes the grid as
the `req` member encoded per the [Jeto grid rules](Jeto.md#grids); a
grid format posts the grid as the whole body, exactly as version 4
does.

### File Upload

Some ops take a file rather than named arguments.  Such an op declares a
single parameter typed as a [sys.files::index] file spec, and the client posts
the file's raw bytes as the body - no argument encoding wraps them.  For
example [repoPublish()] declares its parameter as [sys.files::XetoLibFile]
and is invoked as:

    POST /api/sys/repoPublish
    Content-Type: application/xetolib

    {bytes of the xetolib zip}

The client should send the file's own mime type as the `Content-Type`.
A file parameter is always the op's only parameter, since the body is
the file itself and can carry nothing else.  Upload and
[download](#file-download) are symmetric: a file travels as raw bytes on
the wire in either direction.

## Responses

The `Accept` header selects the response format, or the `xeto-filetype`
query parameter by [filetype name](#filetypes) for easy testing:

    GET /api/demo/readAll?filter=site&xeto-filetype=trio

With no preference the response defaults per version: `text/zinc` for
version 4, `application/json` for version 5.  A format the server cannot
write answers 406.

Version 5 `application/json`, `text/jeto`, and `text/xeto` responses are
the bare result value with no envelope: a function which returns nothing
answers JSON null, or an empty body for xeto.  A Haystack grid format
bridges the result through a grid, so `Accept: text/zinc` on a version 5
call returns the same grid encoding a version 4 client would see.
Version 4 responses are always grids in every format.

Every JSON dialect - jeto, Hayson, and version 3 - is served with
`Content-Type: application/json` no matter which mime type requested it.

Responses honor `Accept-Encoding: gzip`.

### File Download

An op which returns a file serves the file's raw bytes as the response
body, whichever version or method invoked it.  Content negotiation does
not apply: the file is served with its own mime type rather than the
Accept header's, along with an etag and last-modified so clients can
revalidate with a 304, and a `Content-Disposition` naming the file.

## Box Modes

JSON responses default to `box=auto`: a scalar whose plain form would not
decode back is [boxed](Jeto.md#boxed-scalars) as an object naming its own
spec.  The client may select the mode with a mime parameter on the Accept
header:

    Accept: application/json;box=none

| Mode   | Behavior                                            |
| ----   | ----                                                |
| none   | never box; the wire is exactly what schemas describe |
| auto   | box only where the plain form would not decode back |
| all    | box every scalar                                    |

The `box=none` wire is the one JSON Schema validates - see
[Jeto specs](Jeto.md#specs).  It is also what generated clients should
request, since their types come from the schema rather than the wire.
Its one loss is a Ref's display string, which only a box can carry.  An
unrecognized box token answers 406.

## Errors

Version 5 reports every failure as a real HTTP status code with a JSON
body conforming to [sys.api::ApiErr]:

    403 Forbidden
    Content-Type: application/json

    {
      "spec": "sys.api::PermissionErr",
      "status": 403,
      "dis": "Cannot call func: hisWrite",
      "errTrace": "..."
    }

The `spec` names the precise error type from the `sys.api` error
hierarchy, so a client can dispatch on it without parsing the message.
Common mappings:

| Status | Error specs                                                  |
| ----   | ----                                                         |
| [400](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/400) | [sys.api::InvalidArgsErr], [sys.api::UnsupportedVersionErr]  |
| [401](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/401) | [sys.api::AuthErr]                                           |
| [403](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/403) | [sys.api::PermissionErr]                                     |
| [404](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/404) | [sys.api::UnknownFuncErr], [sys.api::UnknownEntityErr], [sys.api::InvalidPathErr], [sys.api::AmbiguousFuncErr] |
| [405](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/405) | [sys.api::MethodNotAllowedErr]                               |
| [406](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/406) | [sys.api::NotAcceptableErr]                                  |
| [415](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/415) | [sys.api::UnsupportedMediaTypeErr]                           |
| [429](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/429) | [sys.api::RateLimitErr] (with `Retry-After` header)          |
| [501](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/501) | [sys.api::NotImplementedErr]                                 |
| [503](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/503) | [sys.api::UnavailableErr]                                    |
| [504](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/504) | [sys.api::TimeoutErr]                                        |

Error bodies are always JSON regardless of the Accept header: an error
must be deliverable even when content negotiation is itself what failed.
Servers may omit `errTrace` in deployments which disable traces.

Every error response carries a `Xeto-Version` header reporting the
server's current version, no matter which version the request selected:
many errors occur before version resolution, so echoing the negotiated
version is not even well defined on the error path.

Version 4 keeps its legacy contract for errors raised by the op itself:
a 200 response carrying an error grid.  Failures in the HTTP processing
before the op still answer HTTP status codes.

## JSON Schema

The `openapi` operation exports the API surface as an OpenAPI 3.1
document whose schemas are generated per [Jeto specs](Jeto.md#specs).
Schemas describe the `box=none` wire, so clients generated from them
should request `application/json;box=none` as described in
[Box Modes](#box-modes).

