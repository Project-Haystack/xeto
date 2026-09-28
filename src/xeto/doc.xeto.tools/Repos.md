# Overview

A *repo* is a database of Xeto libraries.  Repos manage the storage and
distribution of Xeto libraries. The *local repo* is where libs are installed
and used at runtime.  A *remote repo* is a network-accessible source used
to install libs to the local system.  Remote repos are configured by the
user and may be added or removed as needed.

# Local Repo

The local repo manages the set of libs installed on the local platform.
It is the authoritative source for runtime lib resolution: when a
namespace loads a lib, it reads from the local repo.

The local repo is organized on the file system by the [path](Setup.md#env-path).
Each directory in the path searches for subdirectories under `src/xeto/`
and `lib/xeto/` to find installed libs.  The following file naming conventions
are used:

```
foreach (pathDir in path):
  {pathDir}/src/xeto/{libNameA}.xetolib
  {pathDir}/src/xeto/{libNameB}.xetolib
  {pathDir}/lib/xeto/{libNameA}.xetolib
  {pathDir}/lib/xeto/{libNameB}.xetolib
  ...
```

For a given lib name the source version is used as the latest
version for namespace resolution.

If a given lib was installed from a remote repo, then an `origin.props` sidecar
file records provenance: the URI of the remote it was fetched from, the SHA-256
digest of the bytes, and the timestamp of the fetch. Origin metadata
is written once at install time and is not modified afterwards.

The local repo is identified by the URI `local:` for diagnostic
purposes.

# Remote Repos

A remote repo is a network-accessible source of Xeto libs. Remotes
are configured on the local machine and used to search and fetch libs.
Each remote has a configured name and a URI. The name is a local
identifier used by the CLI and error messages; the URI identifies
the transport and is recorded in the lib provenance origin file. Names
are stable across machines only by convention; URIs are stable globally.

Remotes are configured in a `etc/xeto/config.props` file in any
level of the [env path](Setup.md#env-path).  Config files may be edited by
hand, but the recommended approach is to use the `xeto remote-*` commands
to manage the remote configuration.

Haxall ships with a built-in remote named `xetodev` used to install libs
from central repository hosted at [https://xeto.dev/](https://xeto.dev/).

# Auth Tokens

Remote repos that require an authentication token are managed by environment
variables.  This is designed to keep secrets out of shell histories and version
control.

Auth tokens always use the environment names "XETO_REPO_{x}" where `x` is
the programmatic repo name or repo type.  For example if you create a repo
named "foo", then the auth token should be stored in the environment
variable named "XETO_REPO_FOO".  You can also configure an auth token for
a type of remote; for example all GitHub repos will look for an auth
token named "XETO_REPO_GITHUB" if it does not first find one for a specific
repo name.

You can set these environment variables using any of the usual
mechanisms in your OS.  Haxall also provides a special way to set them
using your [path](Setup.md#env-path) in the "fan.props" file as follows:

```
env.XETO_REPO_GITHUB=my-pat
```

You can use the CLI commands `remote-login` and `remote-logout` to manage
environment variables in "fan.props".  For GitHub repos you will need to
generate a *personal access token* (see GitHub docs).

# Command Line

The `xeto` command line tool provides a set of subcommands for managing
and querying your local and remote repos.

## repo

Use the `repo` command to query the local repo for the installed libs:

```
xeto repo            // list all libs as table
xeto repo sys        // list specific lib
xeto repo sys ph     // list multiple libs
xeto repo sys -full  // list full details of a specific lib
xeto repo -full      // list full details of all libs
```

## remote-list

List all configured remotes repos:

```
xeto remote-list  // list the configured remote repos
xeto rl           // use command alias
xeto rl -pathDir  // include where config is defined in path
```

## remote-add

Add a new remote to the configuration with name and URI:

```
xeto remote-add <name> <uri>
xeto remote-add acme https://acme.com/
```

The name must be a valid tag name (start lower case and contain only
ASCII letters, digits, and underbar).  By default the remote is
configured in the work directory of your path.  Use the `-pathDir`
to configure it at a different level of your path.

## remote-remove

Remove a remote from the configuration. Installed libs whose origin
points at the removed remote are not affected, but they cannot be
updated until a remote with the same URI is configured again.

```
xeto remote-remove <name>
xeto remote-remove acme
```

## remote-login

Use this command to add an [auth token](#auth-tokens) to your fan.props:

```
xeto remote-login            // configure token for default repo
xeto remote-login -r acme    // configure token for repo named 'acme'
xeto remote-login -t github  // configure default token use for all github repos
```

## remote-logout

Use this command to remove an [auth token](#auth-tokens) from your fan.props:

```
xeto remote-logout            // remove token for default repo
xeto remote-logout -r acme    // remove token for repo named 'acme'
xeto remote-logout -t github  // remove default token use for all github repos
```

## remote-ping

Verify network connectivity to a remote repo by name.

```
xeto remote-ping  // ping default remote repo
xeto rp           // using command alias
xeto rp -r acme   // ping remote repo named 'acme'
```

## remote-search

Search for libs on a remote. Accepts a free-text query; the server
ranks results by relevance.

```
xeto remote-search foo  // search for 'foo' in default remote repo
xeto rs foo             // using command alias
xeto rs -r acme foo     // search for 'foo' in repo named 'acme'
```

## remote-versions

List the version details of a specific lib available on a remote.
Versions are always returned latest to oldest.

```
xeto remote-versions foo     // all versions of foo
xeto rv foo                  // using command alias
xeto rv foo -r acme          // from repo named 'acme'
xeto rv foo -limit 10        // increase limit to 10
xeto rv foo -versions 3.1.x  // match version constraints
xeto rv foo-3.1.x            // convenience for above
```

## remote-fetch

Download the xetolib file for a specific lib version from a remote
without installing:

```
xeto remote-fetch foo     // fetch latest version of foo to cwd
xeto rf foo               // using command alias
xeto rf foo-1.2.3         // fetch specific version of foo
xeto rf foo -dir someDir  // fetch to a specific directory
```


## install

Install one or more libs from a remote repo into the local repo.  Libs
are specified as a simple name to install the latest version, or with
a version constraint such as `foo-3.0.7` or `foo-3.0.x`.  Dependencies
are resolved recursively and installed too.

The command first prints its plan as a table of actions - which libs
will be installed at which versions and which were pulled in
transitively - then prompts for confirmation before making any
changes.  Use `-preview` to print the plan without executing it, or
`-y` to skip the confirmation.

If a lib is already installed then install fails; use `update`
instead.  If a dependency requires updating a currently installed lib,
install fails unless you pass the `-upgrade` flag to allow it.

Fetched files are verified against the digest advertised by the remote
catalog, then installed to `lib/xeto/` in your workDir along with an
[origin](#local-repo) sidecar file recording their provenance.

```
xeto install foo     // install latest version of 'foo' from default remote repo
xeto i foo           // using command alias
xeto i foo-3.0.7     // install specific version
xeto i foo-3.0.x     // install with depend wildcards
xeto i foo bar baz   // install multiple libs
xeto i foo -r acme   // install from remote repo named 'acme'
xeto i foo -preview  // dry run preview only
xeto i foo -y        // skip confirmation
xeto i foo -upgrade  // update installed libs if needed to meet foo depends
```

## update

Update installed libs to newer versions.  Each lib is updated from the
remote repo recorded in its [origin](#local-repo) provenance, so there
is no repo option.  Dependencies are resolved recursively just like
install, and the update is rejected if the new version would break the
version constraints of any other installed lib.

The command follows the same plan/confirm flow as install with the
`-preview` and `-y` options.

```
xeto update foo      // update to latest version of 'foo'
xeto u foo           // using command alias
xeto u foo -preview  // dry run preview only
xeto u foo -y        // skip confirmation
```

## uninstall

Remove one or more libs from the local repo.  The lib's xetolib file
and its origin sidecar file are deleted.  Uninstall is rejected if the
lib is a source lib or if any other installed lib depends on it.

The command follows the same plan/confirm flow as install with the
`-preview` and `-y` options.

```
xeto uninstall foo           // remove 'foo' from local repo
xeto uninstall foo bar baz   // remove multiple libs from local repo
xeto uninstall foo -preview  // dry run preview only
xeto uninstall foo -y        // skip confirmation
```

## publish

Publish a xetolib file to a remote repo.  You may publish a lib by
name, a single xetolib file, or a directory of xetolibs, in which case
every xetolib in the directory is published in dependency order over a
single session.  A lib name publishes its xetolib from the local repo:
for a source lib that is the zip produced by `xeto build`.  Files are
loaded and validated locally before any network traffic, so a corrupt
file fails fast.

If no [auth token](#auth-tokens) is configured for the repo, then publish
logs in thru your browser: the command opens the repo's login page and
continues once you have authenticated.  Each run requires a new login.

```
xeto publish foo                   // publish xetolib of lib named 'foo'
xeto publish foo.xetolib           // publish to default repo
xeto publish foo.xetolib -r acme   // publish to repo named 'acme'
xeto publish foo.xetolib -preview  // report without publishing
xeto publish someDir/              // publish whole dir in depends order
```
