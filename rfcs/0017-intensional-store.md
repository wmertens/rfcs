---
feature: intensional_store
start-date: 2017-08-11
author: Wout.Mertens@gmail.com
co-authors: (find a buddy later to help our with the RFC)
shepherd-team: Shea Levy, Vladimír Čunát, Eelco Dolstra, Nicolas B. Pierron
shepherd-leader: Shea Levy
related-issues: (will contain links to implementation PRs)
---

# Intensional Store

## TODO / to explain

- flesh out Trust DB locations and updating/querying/merging multiple
- query service
- efficient distribution of build trace entries
- explain the benefits of late binding and how it improves installs on low-power systems
- script that migrates an existing `/nix/store` closure to `/var/lib/nix`, see Migration

## Summary

This RFC builds on content-addressed derivations ([RFC 62], the `ca-derivations` experimental feature) to maximize their benefits.

### What Nix already provides

These are not part of this RFC, they are mentioned because the rest of the RFC relies on them:

- Floating content-addressed derivation outputs. They are built under a scratch path, scanned for references and rewritten to their final path. Self-references are replaced by a sentinel value while hashing. See [building].
- Content-addressed store paths whose digest covers the file system object, the references, a self-reference flag, the store directory and the name. See [store path calculation].
- Content-addressed store objects are trusted without signatures, so they can be substituted from any cache. See `nix store make-content-addressed`.
- The [build trace] (formerly "realisations") maps a resolved derivation output to a content-addressed store path. Entries are signed, and binary caches serve them at `build-trace-v2/<drvName>/<outputName>.doi`.
- Binary caches serve the objective metadata of a store object as a [`.narinfo`][narinfo] file, including `References` and `CA`.
- `nix store verify` and `nix store repair`.
- Hard-linking identical files via `/nix/store/.links` (`nix store optimise`).
- Per-user profiles under `$XDG_STATE_HOME/nix/profiles`, and GC roots including auto roots.
- Configurable store, state and log directories per store (`local?store=…&state=…&log=…`), [chroot stores][local store] and the [local overlay store].
- Single-user installs without `nix-daemon`.

### What this RFC adds

- Decouple subjective metadata (Trust DB) from the Store, keep it per user, merge it from multiple sources
- Move the Store to `/var/lib/nix`
- Store objects provide their objective metadata in-band, next to the entry, so the Store needs no database
- Store can be shared read-write on a network share, with atomic installation via `rename`
- `nix-daemon` becomes optional, also for multi-user installs
- Coordinated garbage collection for shared stores

### Benefits

By making the Store self-describing, we can:

- make the Nix store network-writeable and world-shareable
- verify store paths without access to the Nix Store DB
- keep subjective metadata per user, from multiple trusted sources
- detect non-reproducible builds by comparing build trace entries between sources
- easily switch between single- and multi-user setup

Additionally, this is an opportunity to move the Nix store to a filesystem location supported by most non-NixOS systems, namely `/var/lib/nix`.

By "cleaning up" the filesystem state of Nix, a host of possibilities emerge:

- Boot a cloud VM to a specific system by passing a `$cas` name for stage2. The stage1 will auto-download the stage2 if it's missing and switch to it.
- Cross-compiling can generate `$cas` entries that are reused for native compiles via the build trace. This is useful on low-resource platforms.
- The Nix store doesn't require any support or metadata. On embedded systems, all management of the store can be performed outside the system.
- References to `$cas` entries, such as profiles, are no longer tied to a single system.
- A FUSE filesystem could auto-install `$cas` entries as they are referenced, hanging the I/O until the entry is downloaded and verified.
- You can copy a store from some other install, and immediately use profiles without having their metadata.
- Different Nix tooling and metadata implementations can use the same store

… and so on. Decoupling systems brings exponential possibilities.

### Drawbacks

There are some small drawbacks:

- Garbage collection is more complex when the store is shared between hosts.
- Since the store directory is part of every store path digest, the same content gets a different `$cas` under `/var/lib/nix` than under `/nix/store`. Binary caches need to serve both for the transition period.
- `$cas` entries without metadata are opaque, and might contain malware or illegal content. If nothing references it, there is no problem with the content. Garbage collection takes care of unused entries.
- A hash collision would allow inserting malware into a widely used `$cas`. This is already possible today, but trusting the hashes may lead to wider cache use. Remedies include using secure hashes, scanning for malware, using multiple hashes and comparing between binary caches, …

Note that Nix already assumes that a floating content-addressed build doesn't leak its scratch path into the output, and already signs build trace entries against malicious mappings. This RFC doesn't change that.

### Terminology

We use the Nix concepts, with these shorthands:

- `$drv^out`: a derivation output, meaning a resolved derivation path plus output name. This is the key of a build trace entry.
- `$cas`: the base name `$digest-$name` of a content-addressed store path, as calculated by Nix.
- `$digest`: the hash part of `$cas`.
- Trust DB: a build trace plus subjective metadata, per trusted source.

We assume the following process when wanting to install a given package attribute `$attr`:

- Nix evaluates the desired expressions and determines that a certain derivation output `$drv^out` is required
- `$drv` is [resolved][resolution], which itself looks up build trace entries of its inputs.
- `$drv^out` is looked up in the Trust DB, to possibly yield `$cas`.
- If `$cas` is known:
  - If `$cas` is present in the store, `$attr` is already installed; Done.
  - If `$cas` is present on a binary cache, it is downloaded to the store, without need for a signature; Done.
- `$drv^out` is built using the normal mechanisms for floating content-addressed outputs.
- The resulting build trace entry and subjective metadata are stored in the Trust DB; Done.

Nix already allows a given `$drv^out` to produce different `$cas` entries over time, for example for non-deterministic builds. Each source simply has its own build trace entry.

## Nix Store

## FHS compatibility

Since we're working on the store layer, we have the opportunity to split up the current `/nix` directory and make it FHS compliant. This makes it easier to use Nixpkgs where creating `/nix` is not possible.

An [informal discussion](https://discourse.nixos.org/t/nix-var-nix-opt-nix-usr-local-nix/7101) concluded that the Store should be located at `/var/lib/nix` for maximum compatibility.

Nix chroot stores already allow a store at another physical location, but they require mount and user namespaces and only work on Linux. Changing the logical store directory is also already possible, but loses the binary caches. This RFC makes `/var/lib/nix` the default instead, so binary caches serve it too.

As for the contents of `/nix/var`, all of it can go elsewhere:

- `/nix/var/log` should go under central or per-user log.
- `/nix/var/nix`:
  - `db`: Store database. Objective metadata moves into the Store (see Metadata), subjective metadata into Trust DBs.
  - `daemon-socket`: Builder service. Should move to appropriate location for service sockets, like `/run`
  - `gc`: See the Garbage Collection section
  - `gcroots`, `profiles`: system profiles and roots go under `/var/lib/nix-profiles`. User profiles already live under `$XDG_STATE_HOME/nix/profiles`.
  - `temproots`, `userpool`: Builder service. Should move to appropriate locations for services, like `/var/tmp` and `/var/lib`

Nix already supports setting these per store, so this is a change of defaults.

### Contents

The Store should be verifiable, and only contain verifiable paths. However, to allow atomic installation over the network, there should be a directory for staging an installation. Some other operations also need supporting directories.

For cosmetics and wildcard expansion, we hide supporting directories from regular view.

Therefore, these are the Store contents, all part of the same mount point to ensure atomic semantics:

- `$cas`: a self-validating store object. Any path matching the store path format is subject to verification at any time, and is moved to `.quarantaine` if verification fails
- `$digest.narinfo`: the objective metadata of `$cas`, see Metadata
- `.prepare`: this directory can be used by anyone to prepare a store object before adding it to the Store, by picking a non-conflicting subpath
- `.stage`: after preparing, the store object is moved here
- `.daemon`: if there is a store daemon, it might use this path to prepare installation
- `.quarantaine`: whenever a non-compliant path is encountered, it is moved here
- `.links`: used to hard-link identical store files, as Nix already does
- `.gc`: used to communicate about garbage collection
- anything else doesn't belong in the Store and should be removed

The timestamps of files/directories are kept 0, and the user and group ownership are recommended to be a single user, for example `root:root` or `store:store`.
Note that for a shared store, two systems might see different ownership values; this is acceptable.

### Metadata

For a given `$cas` entry, there is objective and subjective metadata.

Objective metadata examples:

- `$cas`
- content address (the `CA` field)
- NAR hash and size
- references (runtime dependencies)
- late binding information

Subjective metadata examples:

- `$drv^out`
- version
- build timestamp and duration
- build-time dependencies
- nixpkgs commitish (not always applicable)
- name of the attribute in nixpkgs
- own configuration commitish
- list of flakes involved

Objective metadata is a function of the entry contents only and can be calculated by anyone, with the exception of references (see below).

Subjective metadata is a function of the build description and execution. It depends on the source, is trusted data, and any desired metadata should be stored in the Trust DB.

#### In-band metadata

Nix keeps the objective metadata in its SQLite database, and binary caches serve it as `.narinfo` files. Neither is available when all you have is the store directory.

Runtime dependencies are the important case. You can't always detect them based on the contents of the entry, and in order to know which Store entries belong together, they are a necessity. Nix already includes them in the store path digest, so for a given `$cas` there can be only one correct list.

Therefore, every entry comes with a file `/var/lib/nix/$digest.narinfo`, in the `.narinfo` format. It only contains objective fields:

- `StorePath`
- `NarHash`
- `NarSize`
- `References`
- `CA`

Other fields, like `Deriver` and `Sig`, are subjective and belong in the Trust DB. `URL`, `Compression`, `FileHash` and `FileSize` describe a binary cache download and don't apply.

The file validates itself: the store path is calculated from `CA`, `References`, the store directory and the name, and must equal `StorePath`. If the file is missing or altered, the `$cas` won't validate.

Note that this makes the Store directory look a lot like a binary cache, minus the compression. This is on purpose.

Other objective metadata that could be useful, such as late binding information, could be added as extra fields. This is to be determined.

### Trust DB

The Trust DB contains the build trace entries of a source, plus subjective metadata. Each entry contains:

- `$drv^out`
- `$cas`
- signatures
- optional:
  - version
  - description
  - build timestamp
  - builder
  - custom metadata, like one or more git repo commit hashes

With this information, a user can quickly find `$cas` entries to install that match a name or description. `nix-build` can find a `$cas` by `$drv^out`.

Nix currently keeps the build trace in the store database, per store. Here we keep it per user and per source instead. For a given `$drv^out`, there can be many entries, one for each trusted source. This can be handled by having one SQLite DB per source (including localhost), and having an order of precedence.

Note that Nix doesn't garbage collect the build trace yet. Per-source Trust DBs make that a per-user policy decision.

### Sharing the Nix Store

Since the Nix Store (minus supporting directories) contains only self-validating paths, it can be shared "infinitely", only limited by:

- disk space
- network performance
- confidence around hash collision attacks
- confidence around writers corrupting paths without detection

The installation step only involves moving a proposed path from `.prepare` to `.stage`, so no further communication is necessary with the Store daemon.

For single-user installs, the Store can trivially be maintained by the Nix tools, and converting to multi-user is only a matter of changing the permissions.

The Nix local overlay store already allows layering a local store on a shared read-only one. A shared read-write Store goes further, since any host can install into it.

It would even be possible to use FUSE to automatically download any paths that are referenced in the Store, hanging the I/O request while it's being downloaded.

### Store Daemon

Optionally, a daemon can maintain the Store. In this case, it is recommended be the only user with write access. It performs installations, verifications and garbage collection, described below.

### Preparing

Nix already builds floating content-addressed outputs and turns them into store objects, see [building]. That process stays as is, except that the result is written to `.prepare` instead of being registered in a database.

After preparing, Nix writes `$digest.narinfo` next to the entry, and adds the build trace entry and metadata to the user's Trust DB.

Store objects can also come from elsewhere, for example `nix store add` or a substitution. They follow the same steps.

### Installation

We use rename semantics to provide atomic installations. Prepared `$cas` entries are moved to their final location with a `rename` call, which is atomic but requires the path to be on the same filesystem.

Atomicity is important to ensure that `$cas` entries are always valid. If they are copied instead, they don't self-validate for the duration of the copy.

#### with Store Daemon

Any user with write access to `/var/lib/nix/.prepare` and `/var/lib/nix/.stage` can ask for entries to be installed. To do so:

1. They prepare entries in `/var/lib/nix/.prepare`, each as `$cas` and `$digest.narinfo`.
1. They atomically move prepared paths to `/var/lib/nix/.stage`, in reverse dependency order, meaning dependencies of an entry are moved first. First the `$digest.narinfo` file is moved and then the `$cas` entry.

When the Store daemon discovers a new `$cas` entry under `.stage`:

1. If the Store already contains this `$cas` entry, it removes this new one, perhaps first verifying the Store copy.
1. It recursively changes ownership of `$cas` and `$digest.narinfo` to itself and timestamps to 0, making sure that write permission is removed for everybody, and read permission is added for anybody.
   If it has no permissions to do this, it instead copies the path into `/var/lib/nix/.daemon`, and another process will need to keep `.stage` clean.
1. The daemon verifies the `$cas`. If it doesn't match, it removes `$cas` and `$digest.narinfo`. Note that a missing or altered `$digest.narinfo` file won't pass validation.
1. It checks that all references are already present in the Store. If not, the path is held for a while and deleted if the references don't appear in time (configurable).
1. It atomically moves `$digest.narinfo` into `/var/lib/nix`.
1. It atomically moves `$cas` into `/var/lib/nix`.

Note that to ensure atomicity, `.prepare` and `.stage` need to be on the same filesystem, and either `.stage` or `.daemon` need to be on the same filesystem as the Store.

#### without Store Daemon

Any user with write access to `/var/lib/nix/.stage` and `/var/lib/nix` can install entries. To do so:

1. They prepare entries in `/var/lib/nix/.stage`, each as `$cas` and `$digest.narinfo`.
1. They atomically move prepared entries to `/var/lib/nix`, in reverse dependency order, meaning dependencies of an entry are moved first, and `$digest.narinfo` is moved before `$cas`

Note that to ensure atomicity, `.stage` needs to be on the same filesystem as the Store.

Note that when two writers are trying to install the same `$cas` or `$digest.narinfo`, one of them might get an error, but the end result will be the same (as long as the `$cas` is self-valid). So multiple writers can also be on separate hosts, in a trusted setting.

### Verification

A path in the Store is verified like `nix store verify` does for content-addressed paths, but using `$digest.narinfo` instead of the database. If it doesn't match, the path is moved to `/var/lib/nix/.quarantaine`, where a sysadmin has to investigate.

Any process with write access to `/var/lib/nix` and `/var/lib/nix/.quarantaine` can do this, for example the Store daemon.

### Garbage collection

Garbage collection needs to identify store paths that are not used by anything on any of the systems sharing the same store. Here we propose a simple mechanism for coordination, but any mechanism is acceptable.

- A host with store write access decides to run garbage collection.
- It checks that `.gc/running_gc` does not exist or contains a very old timestamp, and writes a unique number to `.gc/will_gc`.
- After waiting long enough to prevent collisions (for example 10 seconds), it reads `.gc/will_gc` and verifies it contains the unique number it wrote.
- It clears out `.gc/` except for the file `.gc/will_gc` and adds the file `.gc/running_gc` containing the current timestamp.
- While it waits for other hosts, it checks the Store for `$digest.narinfo` files that don't have a matching `$cas`.
- Each host's store daemon monitors `.gc/running_gc` at some interval, for example 1 minute.
- While this file exists, the daemon must record its root `$cas` entries, by creating 0-length files named `.gc/$cas`.
- The writer waits long enough for all the hosts to record their GC roots, for example 10 minutes.
- It verifies that `.gc/will_gc` still contains its unique number
- After the wait period expired, the writer host scans for store paths that are not part of the own and other GC roots. Each `$cas` is atomically moved to `.gc` and deleted; `$digest.narinfo` is also deleted.
- The `$digest.narinfo` files that still don't have their matching `$cas` are removed. Note that when installing, the `$digest.narinfo` will appear shortly before `$cas` since everything is prepared.
- Finally, the writer host empties the `.gc` directory, leaving the `running_gc` file for last.

For a single-user installation or a non-shared Nix store, none of this is necessary, and the GC process remains unchanged, except for the new locations to search for GC roots.

## Profiles

Nix profiles and GC roots stay as they are, see [profiles]. User profiles already live under `$XDG_STATE_HOME/nix/profiles`. System profiles and roots move from `/nix/var/nix` to `/var/lib/nix-profiles`.

Since a profile points to an immutable `$cas` path, it is the same across systems and can therefore be part of a network-mounted home directory.

However, a profile link itself is trusted information, and should be shared between users and systems only when they trust each other.

For a shared store, the GC roots of each host are recorded as described in Garbage collection.

## Administration Tasks

### Migration

There is no real need for migrating stores, since `/nix/store` and `/var/lib/nix` can coexist and the tooling either uses one or the other. However, it is convenient to migrate built artifacts for implementing this RFC.

Nix already converts a closure to content-addressed form with `nix store make-content-addressed`, but only within the same store directory. Moving to another store directory means rewriting every reference.

To migrate an existing input-addressed path `/nix/store/$old` to `/var/lib/nix/$cas`, the following approach will work most of the time:

- migrate all its dependencies using the below steps
- replace all strings of the form `/nix/store/$old` with `/var/lib/nix/$cas`, and calculate `$cas` as Nix does for content-addressed paths, with self-references replaced by the sentinel
- do the same with symlinks, but consider relative paths as well
- write `$digest.narinfo` and place the entry in `/var/lib/nix/$cas`

Note that `/var/lib/nix` is 2 characters longer than `/nix/store`, while the digest has the same length. To keep binaries patchable in place, the name has to be 2 characters shorter, for example by dropping the version suffix or truncating. This is to be determined.

This process will fail if the store object refers to the Store in ways that aren't visible, like different string encoding and calculated paths. Rebuilding with content-addressed derivations avoids this, at the cost of a full rebuild.

### Adding a Store Daemon

To begin managing an existing Store with a Store Daemon, these steps are performed:

- Change permissions on the Store root so only the daemon has write access.
- Ensure `.prepare`, `.stage` and `.quarantaine` with desired permissions.
- For each Store entry
  - Recursively adjust permissions and timestamps
  - Verify entry
    - If invalid, move to `.quarantaine` and try to download replacement from known caches

### Removing a Store Daemon

- Wait for pending installs to complete.
- Stop Store Daemon.
- Change permissions on the Store as desired.

### Repairing an entry

As Nix already does with `nix store repair`. Since `$cas` entries need no signature, any cache that has it will do.

## Implementation

- NixPkgs needs to be audited to remove hard-coded `/nix` names, replacing it with the store path variable (TODO look up name).
- Nix needs a store type that reads objective metadata from `$digest.narinfo` instead of SQLite, and keeps the build trace in per-user Trust DBs. The tools either use the old location and semantics, or the new one.
- Binary caches already serve `.narinfo` files and build trace entries. They need to serve `/var/lib/nix` paths as well.
- Build trace entries need to be distributed in an incremental way. For example, as a JSON array of added and changed entries since some timestamp.

## Alternative options

There are a few choices made in this RFC, here we describe alternatives and why they were not picked.

### Keep store at `/nix/store`

The `$cas` entries and `$digest.narinfo` files could stay in `/nix/store`. The benefit would be that NixPkgs doesn't have to be audited for hardcoded `/nix` paths, existing binary caches keep working, and migration doesn't need shorter names.

However, this keeps the problem of some installations not having permission to create a `/nix` directory. Chroot stores work around that, but only on Linux. It also makes it much harder to share the store between hosts (as long as input-addressed entries are present).

### No metadata

Not keeping metadata in the Store means that the Store by itself doesn't have enough information to do garbage collection, nor to let a system boot from a given `$cas`.

### Keep metadata in directory with Store entry

Since the Store entries can be files or directories, that means that files would have to be put in a directory, for example `$cas` becomes `$cas/_`.
Then directory entries would have to do the same for symmetry. This requires many code changes and requires extra storage, even if an entry doesn't have any runtime dependencies.

### Remove the name from store paths

An earlier version of this RFC used only the hash as store path. This makes the store more opaque and requires good tooling for manual management. Nix includes the name in content-addressed store paths, and we follow Nix.

[RFC 62]: https://github.com/NixOS/rfcs/blob/master/rfcs/0062-content-addressed-paths.md
[building]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/building.md
[store path calculation]: https://github.com/NixOS/nix/blob/master/doc/manual/source/protocols/store-path.md
[build trace]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/build-trace.md
[resolution]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/resolution.md
[narinfo]: https://github.com/NixOS/nix/blob/master/doc/manual/source/protocols/binary-cache/narinfo.md
[local store]: https://github.com/NixOS/nix/blob/master/src/libstore/local-store.md
[local overlay store]: https://github.com/NixOS/nix/blob/master/src/libstore/local-overlay-store.md
[profiles]: https://github.com/NixOS/nix/blob/master/doc/manual/source/command-ref/files/profiles.md
