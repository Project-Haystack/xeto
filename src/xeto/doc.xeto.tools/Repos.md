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
  {pathDir}/src/xeto/{libNameA}/
  {pathDir}/src/xeto/{libNameB}/
  {pathDir}/lib/xeto/{libNameA}/{libNameA}-{version1}.xetolib
  {pathDir}/lib/xeto/{libNameA}/{libNameA}-{version2}.xetolib
  {pathDir}/lib/xeto/{libNameB}/{libNameB}-{version1}.xetolib
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

# Command Line

The `xeto` command line tool provides a set of subcommands for managing
and querying your local and remote repos.

## repo

Use the `repo` command to query the local repo for the installed libs:

```
xeto repo                  // list the latest version of all libs
xeto repo sys -versions    // list all versions of the sys lib
xeto repo -full            // list full details of latest version of all libs
xeto repo -full -versions  // list the latest version of all libs
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
to configure it a different level of your path.

## remote-remove

Remove a remote from the configuration. Installed libs whose origin
points at the removed remote are not affected, but they cannot be
updated until a remote with the same URI is configured again.

```
xeto remote-remove <name>
xeto remote-remove acme
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

