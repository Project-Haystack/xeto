# Overview

[xeto.dev](https://xeto.dev/) is the central repository for publishing and
distributing Xeto libraries.  This chapter walks through the full process
to publish a library there for the first time.

Publishing requires three things, in order:

1. An **organization** that owns your libraries - [request one](#requesting-an-organization)
2. A lib name **prefix** registered to that organization - [request one](#requesting-a-prefix)
3. A built **library** published under that prefix - [publish it](#publishing)

An *organization* (org) is the account that owns lib prefixes and the
libraries published under them.  The person who requests an org becomes its
admin on approval and can add other members.

A *prefix* is the leading portion of a lib name, such as `acme` or
`com.acme`.  Prefixes guarantee globally unique lib names and are registered
to exactly one org.  Whoever owns the prefix controls who may publish
libraries whose names fall under it.  For example, an org that owns `acme`
may publish `acme`, `acme.assets`, `acme.points`, and so on.

Org and prefix requests are each reviewed by a person before anything is
created.  Each request is tracked as a ticket, and you receive email as it
moves through review.  Once your org owns a prefix, publishing is self-service
through the CLI.  You can later manage published libs - see [Managing Published Libs](#managing-published-libs).

Before you start:

- Create an account and sign in at [xeto.dev](https://xeto.dev/).  Requesting
  an org or a prefix requires a signed-in session.
- Install the Xeto CLI tools, which you will use to build and publish.  See
  [Setup](Setup.md).

# Requesting an Organization

Before you can register a prefix or publish, you need an org.  If your
company or project already has an org on xeto.dev, ask one of its admins to
add you as a member instead of requesting a new one.

To file the request:

1. Sign in at [xeto.dev](https://xeto.dev/).
2. Click your username at the top of the page to open your account **Settings**.
3. In the **Organizations** section, click **Request a new organization**.
4. Fill in the request form:
   - **Organization name** - the display name, such as `Acme Inc`
   - **Email domain** - your organization's domain, such as `acme.com`
   - **Mailing address** - a postal address for the organization
5. Click **Check**.  If the name or the domain is already taken by another
   org, the check is rejected and links you to the org that holds it.
   Otherwise it reports that the request is available and will go to human
   review.
6. Click **Submit Request**.

Editing the name or domain after a check clears the result, so you re-check
before submitting again.

## Org Review

A submitted request moves through these states:

    open  ->  review  ->  approved  (or  denied)

A reviewer may comment on the ticket. Comments are received by email and you
can reply on the ticket page.  When the request is **approved**, the org is
created and you become its admin, with permission to manage members and to
administer its libraries.

The **Request a new organization** page lists the requests you have filed
and the current status of each.  Click a request to open its ticket.

## Adding Members

Open your account **Settings**, click the org name in the **Organizations**
section, then click **Manage** on the org page.  Under **Members**, an org
admin can add other users by email and grant them permissions, including the
ability to publish libraries under the org's prefixes.

# Requesting a Prefix

Registering a prefix reserves it for your org and lets you publish any
library whose name falls under it.  Read
[Libs](../doc.xeto/Libs.md#lib-name-prefixes) for the full naming conventions
before you choose one.

## Choosing a Prefix

The request is validated against these rules:

- The prefix must be a valid lib name.
- It may have at most two dotted sections, such as `acme` or `com.acme`.
- Each section must be at least two characters.
- It must not already be registered to another org.

Common forms:

- **Top level**, such as `acme` - granted to vendors or projects when
  appropriate to avoid name squatting.
- **Reverse DNS**, such as `com.acme` - the first section must be a
  registered base such as a top level domain (`com`, `org`, `io`, `net`, and
  so on).  A name matching your org's email domain (`com.acme` for an org
  whose domain is `acme.com`) is strong evidence for the reviewer.
- **Community contribution**, such as `cc.baz` - use the `cc.` base for a
  community contribution tied to a vendor named `Baz`.

Reserved words can still be submitted, but the check warns you that they are
likely to be denied unless there is a clear reason to grant them.

## Filing a Prefix Request

1. Sign in at [xeto.dev](https://xeto.dev/).
2. Click your username at the top of the page to open your account **Settings**.
3. In the **Organizations** section, click your org, then click **Manage** on
   the org page (visible to org admins).
4. Under **Prefixes**, enter:
   - **Lib prefix** - the prefix you want, such as `acme`
   - **Justification** (optional) - why your org should own this prefix
5. Click **Check**.  A rejection explains what is wrong, such as the prefix
   already being registered; a warning flags a submittable but risky name;
   otherwise the prefix is available to request.
6. Click **Request Prefix**.

Editing the prefix after a check clears the result, so you re-check before
submitting again.

## Prefix Review

A reviewer evaluates every prefix request:

    open  ->  review  ->  granted  (or  denied)

You receive email as the request is commented on, granted, or denied.  When
it is **granted**, the prefix is registered to your org and you may begin
publishing libraries under it.  The org **Manage** page lists the org's
registered prefixes and its open prefix requests.

# Publishing

Once your org owns a prefix, you publish with the Xeto CLI.  The tools ship
with a built-in remote named `xetodev` pointing at
[https://xeto.dev/](https://xeto.dev/), which is the default publish target.
See [Setting Up the CLI](#setting-up-the-cli) to get ready, and
[Repos](Repos.md) for how local and remote repos work.

## Setting Up the CLI

1. Install the Xeto tools for Java or NodeJS - see [Setup](Setup.md).
2. Lay out your source so the tools can find it.  Libs are found on the
   [env path](Setup.md#env-path) under `src/xeto/{lib-name}/`, so a lib named
   `acme.assets` lives at `{workDir}/src/xeto/acme.assets/lib.xeto`.  The
   workDir is found by walking up from your current directory to a
   `xeto.props`, `fan.props`, or `.git` marker.
3. Run `xeto version` from your project directory and check that
   `xeto.workDir` and `xeto.path` include your source tree.
4. Run `xeto remote-list` to confirm the built-in `xetodev` remote is
   configured.

## Naming Your Library

Your lib's name must fall under a prefix your org owns.  The lib name is
determined by the directory name of its source tree, and that directory must
contain a `lib.xeto` pragma file, which must sit on the env path (see
[Setting Up the CLI](#setting-up-the-cli)).  See [Libs](../doc.xeto/Libs.md#source-tree).

In `lib.xeto`, set the `org` metadata so the published lib records who owns it:

```
pragma: Lib <
  doc: "Acme building asset models"
  version: "1.0.0"
  maturity: "alpha"
  org: {
    dis: "Acme Inc"
    uri: "https://acme.com/"
  }
  depends: {
    { lib: "sys", versions: "5.x" }
  }
>
```

See [Meta](../doc.xeto/Meta.md) and [Build](../doc.xeto/Build.md) for the full
set of lib metadata and build variables.

## Build and Publish

Compile your source lib into a xetolib zip, preview the publish, then publish:

```
xeto build acme.assets
xeto publish acme.assets -preview   // dry run, report only
xeto publish acme.assets            // publish to the default xetodev remote
```

Publishing logs you in through your browser: the command opens the xeto.dev
login page and continues once you have authenticated.  Each run requires a
fresh authorization.  See [publish](Repos.md#publish) for the other forms of the
command, such as publishing a whole directory of xetolibs.

## Versioning

Bump the `version` in your `lib.xeto` before every publish.  Each publish is a
new immutable version: the exact coordinates (`name` + `version`) can never be
reused once published, and they stay burned even after the version is yanked
or deleted.  Republishing the same version fails with *Version already
published*.

The version may be a literal such as `"1.0.0"`, or come from a build variable
in `xeto-build.props` shared across a set of libs.  See
[Libs](../doc.xeto/Libs.md#version) and [Build](../doc.xeto/Build.md).  There
is also a per-lib daily publish limit.

## Maturity

Declare a version's maturity with the `maturity` tag in the `lib.xeto` pragma.
There are three levels:

- `alpha` - early experimental prerelease
- `beta` - feature complete prerelease
- `stable` - stable production release

If you omit `maturity` the version is treated as `stable`, so set it
explicitly while a lib is still a prerelease to avoid announcing it as
production ready.

Maturity is locked when a version is published and never changes afterwards.
To move a lib forward you publish a new version with the new maturity, for
example:

```
acme.assets-1.0.0   maturity: "alpha"
acme.assets-1.0.1   maturity: "beta"
acme.assets-1.0.2   maturity: "stable"
```

xeto.dev tracks a lib's latest version and its latest stable version
separately, and `xeto remote-search` shows both.  Publishing a prerelease
advances the latest line but not the stable line.

## Who May Publish

The server authorizes a publish by finding the longest registered prefix that
matches your lib name, then checking that you hold publish permission on the
org that owns it.  An org admin grants that permission on the org's **Manage**
page.

## Common Errors

- **No lib prefix for `{name}`** - no registered prefix matches your lib
  name.  [Request a prefix](#requesting-a-prefix) that covers it.
- **Lib prefix `{base}` requires a two tier grant** - the match is a base
  such as `com` or `org`, which never authorizes its children on its own.  You
  need a two tier prefix such as `com.acme`.
- **Lib prefix `{prefix}` not granted to publisher** - the prefix is owned by
  an org you are not authorized to publish for.  Ask an org admin for publish
  permission.
- **Version already published: `{name}-{version}`** - bump the version.
- **Publish rate limit exceeded** - you have hit the per-lib daily publish
  limit; wait and retry.

# Managing Published Libs

After a lib is published you manage it from its page on
[xeto.dev](https://xeto.dev/).  There are two levels of action: yank and
delete act on a single version, while deprecate applies to the whole lib.  All
of these controls appear only if you hold publish permission on the org that
owns the prefix, and none of them frees a version number for reuse - published
coordinates stay burned.  There is no way to remove a whole lib, but it can be
deprecated - see [why](#why-libs-cannot-be-removed).

## Why Libs Cannot Be Removed

This is deliberate.  Once a lib is published, other libs and projects come to
depend on it, and removing it breaks everything built on top.

xeto.dev is designed so that a lib cannot disappear out from under its users:

- A whole lib can never be removed.  Retire it by
  [deprecating](#deprecating-a-lib) it instead.
- A flawed version can be [yanked](#yanking-a-version).  New resolution skips
  it, but its content stays downloadable so existing pins keep working.
- [Deleting](#deleting-a-version) is limited to a single version within 72
  hours of publishing, before others are likely to depend on it, and the
  version number can never be reused for different content.

## Managing Versions

Yank and delete operate on one version at a time.  To reach their controls,
open the lib's page and click **Versions** (or **All versions**). The **Yank**
and **Delete** buttons are on each row of the version list.  **Delete** is greyed out once the [delete
window](#deleting-a-version) has passed.

### Yanking a Version

Yanking withdraws a single version from new resolution: it drops out of
listings and version ranges, and the lib's latest line recalculates to skip
it.  The version stays downloadable, so anything pinned to that exact version
keeps working.  You may supply a reason that is shown to consumers.  Yank when
a version is flawed but you do not need to remove its content.

In the version list, click **Yank** on the version.  **Un-yank** restores it
so ranges and the latest line resolve it again.

### Deleting a Version

Deleting permanently removes a version's published content: every pinned
consumer breaks, and the version number stays burned so it can never be
republished.  This cannot be undone.  Prefer yanking unless the content itself
must be removed, such as an accidental secret or private data.

The owning org may delete a version only within 72 hours of publishing it. In
the version list, click **Delete**, optionally give a reason, and type today's
date to confirm.

### Yank or Delete?

| Aspect             | Yank           | Delete |
| ------------------ | -------------- | ------ |
| Content stays      | yes            | no     |
| Resolves by range  | no (pins only) | no     |
| Pins keep working  | yes            | no     |
| Reversible         | yes, un-yank   | no     |

## Deprecating a Lib

Deprecation applies to a whole lib rather than one version, steering consumers
away with a message shown on the lib's pages.  It does not remove any version
or block installs - existing and pinned usage keeps working.  You may name a
superseding lib, which must already be published on xeto.dev.  This is the way
to retire a lib, since a whole lib [cannot be removed](#why-libs-cannot-be-removed).

On the lib page, click **Deprecate**, enter the message, and optionally name
the lib that supersedes it.  **Undeprecate** clears it.

Once published, your library appears on [xeto.dev](https://xeto.dev/) with its
documentation, source, and downloads, and others can install it with
`xeto install`.
