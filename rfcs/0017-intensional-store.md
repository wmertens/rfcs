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

- Trust DB locations, per user and system-wide
- the protocol for Trust DB sources: lookups, plus an incremental feed of build trace entries, see Infrastructure
- security considerations: attack scenarios and their mitigations
- check that Nix accepts an empty name in the digest calculation, see Remove the name from store paths
- script that migrates an existing `/nix/store` closure to `/var/lib/nix`, see Incidental improvements
- quantify the savings on a Hydra-sized store, from early cutoff and from FUSE path masking
- find someone to implement it, see Implementation

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
- Store objects provide their objective metadata in-band, next to the entry, so the Store needs no database
- Store can be shared read-write on a network share, with atomic additions via `rename`
- `nix-daemon` becomes optional, also for multi-user installs
- Coordinated garbage collection for shared stores
- Tooling to query the Trust DBs instead of the Store
- Incidental improvements: drop the name from store paths, move the Store to `/var/lib/nix`, provide the Store via FUSE, and make the Store non-listable

### Motivation

Nix's content-addressed derivations already give early cutoff: when a rebuilt dependency turns out to be identical, its dependents don't need rebuilding. However, they don't change how the store itself is managed:

- The store can't be used without its SQLite database, so it can't be shared between hosts, and copying a store means copying its database too.
- The build trace is per store. Adding a substituter or another source of build trace entries affects every user of the store, so only trusted users can do it.
- Adding entries to a multi-user store requires talking to the Nix daemon.

This RFC makes the filesystem the store database, keeps trust per user in a Trust DB, and lets any host add entries to a shared Store with just a `rename`. It is a small step on top of content-addressed derivations, and most of the store layer code carries over, see Implementation.

### Benefits

By making the Store self-describing, we can:

- make the Nix store network-writeable and world-shareable
- verify store paths without access to the Nix Store DB
- keep subjective metadata per user, from multiple trusted sources
- let an untrusted user add their own substituter or build trace source, without affecting anyone else
- detect non-reproducible builds by comparing build trace entries between sources
- easily switch between single- and multi-user setup

By "cleaning up" the filesystem state of Nix, a host of possibilities emerge:

- Boot a cloud VM to a specific system by passing a `$cas` name for stage2. The stage1 will auto-download the stage2 if it's missing and switch to it.
- Cross-compiling can generate `$cas` entries that are reused for native compiles via the build trace. This is useful on low-resource platforms.
- The Nix store doesn't require any support or metadata. On embedded systems, all management of the store can be performed outside the system.
- References to `$cas` entries, such as profiles, are no longer tied to a single system.
- A FUSE filesystem could auto-fetch `$cas` entries as they are referenced, hanging the I/O until the entry is downloaded and verified, see Incidental improvements.
- You can copy a store from some other install, and immediately use profiles without having their metadata.
- Different Nix tooling and metadata implementations can use the same store

… and so on. Decoupling systems brings exponential possibilities.

### Drawbacks

There are some small drawbacks:

- Garbage collection is more complex when the store is shared between hosts. Also, a host or user could root huge amounts of data. Quotas per host and per user are left for later.
- `$cas` entries without metadata are opaque, and might contain malware or illegal content. If nothing references it, there is no problem with the content. Garbage collection takes care of unused entries.
- A hash collision would allow inserting malware into a widely used `$cas`. Nix already assumes collisions are impossible in practice for fixed-output derivations, content-addressed paths and derivation hashes, but trusting the hashes may lead to wider cache use. Therefore the Store only accepts content addresses based on SHA-256, since SHA-1 collisions have been published. Further remedies include scanning for malware, using multiple hashes and comparing between binary caches, …
- Hidden self-references break content-addressed builds. When an output contains its own scratch path in a form that Nix can't find, like a compressed man page, a JAR or a signed binary, the rewrite misses it. The finished entry then points to a path that doesn't exist, and a different derivation building the same content gets a different `$cas`. This is already the case for Nix's content-addressed derivations. Such leaks are detected by building twice with different scratch paths and comparing the results. Note that Nix's scratch path is deterministic, made from the derivation path and the output name, so a second build uses the same one and the leak stays hidden. Therefore the check, for example `--check`, adds a random nonce to the scratch path. The leaks are fixed with rewriters in the build, for example with Nix's IPC builder protocol (`builder-rpc-v0`, in development), where the builder can unpack, rewrite and repack such files itself. References to dependencies don't have this problem, since the build already sees their final `$cas`.

Note that we don't add a fallback, like a symlink from the scratch path to the `$cas`. Such a symlink can't be validated from its contents, and it would hide the bug instead of getting it fixed.

Note that Nix already signs build trace entries against malicious mappings. This RFC doesn't change that.

### Terminology

We use the Nix concepts, with these shorthands:

- `$drv^out`: a derivation output, meaning a resolved derivation path plus output name. This is the key of a build trace entry.
- `$cas`: the base name `$digest-$name` of a content-addressed store path, as calculated by Nix (or just `$digest`, see Incidental improvements).
- `$digest`: the hash part of `$cas`.
- Trust DB: a build trace plus subjective metadata, per trusted source.

We assume the following process when wanting to realise a given package attribute `$attr`:

- Nix evaluates the desired expressions and determines that a certain derivation output `$drv^out` is required
- `$drv` is [resolved][resolution], which itself looks up build trace entries of its inputs.
- `$drv^out` is looked up in the Trust DB, to possibly yield `$cas`.
- If `$cas` is known:
  - If `$cas` is present in the store, `$attr` is already realised; Done.
  - If `$cas` is present on a binary cache, it is downloaded to the store, without need for a signature; Done.
- `$drv^out` is built using the normal mechanisms for floating content-addressed outputs.
- The resulting build trace entry and subjective metadata are stored in the Trust DB; Done.

Nix already allows a given `$drv^out` to produce different `$cas` entries over time, for example for non-deterministic builds. Each source simply has its own build trace entry.

## Case studies

Here we describe how the Store is used in practice. The details of each mechanism are in the sections that follow.

### Single user on macOS or Linux

- **Setup**: the user owns the Store. There is no daemon.
- **Adding entries**: Nix moves prepared entries from `.stage` into the Store directly, see "without Store Daemon".
- **Trust DB**: in the user's home directory, with their own sources.
- **Garbage collection**: as today, the roots are the user's profiles and auto roots.

Compared to today, there is no store database to keep in sync, so the Store can be copied, backed up or restored like any directory. Switching to multi-user later is only a matter of adding a daemon and changing permissions, see "Adding a Store Daemon".

Note that on macOS, creating `/nix` currently requires a separate APFS volume, mounted via `synthetic.conf`. Moving the Store to `/var/lib/nix` would make that unnecessary, see Incidental improvements.

### Containers

- **Setup**: the host bind-mounts its Store into the containers, including `.stage`. The host's daemon owns the Store.
- **Adding entries**: a container prepares entries and moves them into `.stage`. The host's daemon notices them, validates them and moves them into the Store.
- **Trust DB**: each container has its own, for example as part of its image. It doesn't need to trust the host's mappings, nor the other containers'.
- **Garbage collection**: the host records the roots of its containers, for example by giving each container its own `.gc/hosts/$host` directory.

Compared to today, there is no daemon socket to pass into the container, and the container needs no privileges at all. Today, a container either has its own store baked into its image, or mounts the host store read-only and talks to the host's daemon over its socket to add anything.

Since the Store only holds self-validating entries, and the daemon validates a copy of everything a container adds, a malicious container can't corrupt what the others use. At worst it adds entries that nobody references, and garbage collection removes those. Note that this only holds if the container can write to its staging directory and nothing else: not the Store root, and not `.gc`.

### Multi-user system

- **Setup**: the daemon owns the Store, and is the only one with write access to it.
- **Adding entries**: users prepare entries in `.prepare` and move them to `.stage`. The daemon validates them and moves them into the Store, see "with Store Daemon".
- **Trust DB**: per user. The system has its own Trust DB for the system profiles, maintained by `root`.
- **Garbage collection**: as today, the roots are the profiles and auto roots of all users.

Compared to today, an untrusted user can add their own substituter or build trace source without affecting anyone else. Today, untrusted users can only use the substituters that an administrator listed in `trusted-substituters`.

Note that users still share the Store itself. If two users trust different sources for the same `$drv^out`, they might get different `$cas` entries, and both are stored. This is on purpose: each user only uses the entries they trust.

### Hosts sharing a network Store

- **Setup**: multiple hosts mount the same Store over the network, for example a CI farm, a lab or a cluster. Each host runs a daemon.
- **Adding entries**: as for a multi-user system, on each host. Since additions are atomic renames and entries are self-validating, hosts don't need to talk to each other. When two hosts add the same entry, one of them gets an error, but the result is the same.
- **Trust DB**: per user and per host, as above. Hosts can also share a source, for example the build trace of the CI farm.
- **Garbage collection**: each host keeps its roots in `.gc/hosts/$host`, and any host can collect, see Garbage collection.

Compared to today, this is new: Nix can't share a writable store between hosts. The local overlay store comes closest, but only shares a read-only lower store.

Note that a build done on one host is immediately available to all the others, without copying. For a CI farm, this means a substitution is just a lookup in the build trace.

## Nix Store

### Contents

The Store should be verifiable, and only contain verifiable paths. However, to allow atomic additions over the network, there should be a directory for staging an addition. Some other operations also need supporting directories.

For cosmetics and wildcard expansion, we hide supporting directories from regular view.

Therefore, these are the Store contents, all part of the same mount point to ensure atomic semantics:

- `$cas`: a self-validating store object. Any path matching the store path format is subject to verification at any time, and is moved to `.quarantaine` if verification fails
- `$digest.narinfo`: the objective metadata of `$cas`, see Metadata
- `.prepare`: this directory can be used by anyone to prepare a store object before adding it to the Store, by picking a non-conflicting subpath
- `.stage`: after preparing, the store object is moved here
- `.daemon`: if there is a store daemon, it might use this path to prepare additions
- `.quarantaine`: whenever a non-compliant path is encountered, it is moved here
- `.links`: used to hard-link identical store files, as Nix already does
- `.gc`: used for garbage collection, holding the GC roots of each host and the entries being removed
- anything else doesn't belong in the Store and should be removed

The timestamps of files/directories are kept at 1, as Nix does, and the user and group ownership are recommended to be a single user, for example `root:root` or `store:store`.
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

Therefore, every entry comes with a file `/nix/store/$digest.narinfo`, in the `.narinfo` format. It only contains objective fields:

- `StorePath`
- `NarHash`
- `NarSize`
- `References`
- `CA`

Other fields, like `Deriver` and `Sig`, are subjective and belong in the Trust DB. `URL`, `Compression`, `FileHash` and `FileSize` describe a binary cache download and don't apply.

An entry is valid when all of these hold:

- the file is named after the digest of `StorePath`, and `StorePath` uses the local store directory
- the store path calculated from `CA`, `References`, the store directory and the name equals `StorePath`
- the content address recalculated from the entry's contents equals `CA`, with self-references masked the way Nix does when adding a path
- `NarHash` and `NarSize` match the entry's contents

Note that the store path only covers `CA`, `References` and the name. `NarHash` and `NarSize` are not part of it, so they are always recalculated, never trusted. If the file is missing or any field is altered, the `$cas` won't validate.

Note that this makes the Store directory look a lot like a binary cache, minus the compression. This is on purpose.

The format is simple `Key: Value` lines, so shell tools can read it too. The new store type parses it strictly: exactly the fields above, each once, and nothing else. Other objective metadata, such as late binding information, can only be added later if it is covered by the digest or can be recalculated from the contents. This is to be determined. Note that Nix's parser currently requires `URL`, so the new store type has to parse these files without it.

When adding an entry, `$digest.narinfo` is moved into the Store before `$cas`, so a present `$cas` always has its metadata. Garbage collection removes leftover `$digest.narinfo` files.

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

With this information, a user can quickly find `$cas` entries to realise that match a name or description. `nix-build` can find a `$cas` by `$drv^out`.

Note that the build-time dependencies don't need storing: `$drv` is resolved, so it already lists its inputs as store paths.

Nix currently keeps the build trace in the store database, per store. Here we keep it per user and per source instead. For a given `$drv^out`, there can be many entries, one for each trusted source. This can be handled by having one SQLite DB per source (including localhost), and having an order of precedence.

A source can be a service that answers lookups, like a binary cache serving build trace entries, or a static mapping, like a downloaded SQLite file.

#### Maintenance

Nix doesn't garbage collect the build trace yet. With per-source Trust DBs, this becomes simple:

- The DBs of remote sources are caches. Their entries can be dropped at any time, or the whole DB can be replaced by a newer download.
- The localhost DB keeps the entries whose `$cas` is present in the Store, plus the entries of recent builds, for example of the last month.
- The collector can't write to the Trust DBs of other users. Instead, each user's Trust DB is cleaned up the next time that user runs Nix, by dropping entries whose `$cas` is gone.

### Sharing the Nix Store

Since the Nix Store (minus supporting directories) contains only self-validating paths, it can be shared "infinitely", only limited by:

- disk space
- network performance
- confidence around hash collision attacks
- confidence around writers corrupting paths without detection

Adding an entry only involves moving a proposed path from `.prepare` to `.stage`, so no further communication is necessary with the Store daemon.

For single-user installs, the Store can trivially be maintained by the Nix tools, and converting to multi-user is only a matter of changing the permissions.

Note that the Store only holds content-addressed entries, so input-addressed paths have to be converted or removed first, see Migration.

The Nix local overlay store already allows layering a local store on a shared read-only one. A shared read-write Store goes further, since any host can add entries to it.

It would even be possible to use FUSE to automatically download any paths that are referenced in the Store, see Incidental improvements.

### Store Daemon

Optionally, a daemon can maintain the Store. In this case, it is recommended be the only user with write access. It performs additions, verifications and garbage collection, described below.

### Preparing

Nix already builds floating content-addressed outputs and turns them into store objects, see [building]. That process stays as is, except that the result is written to `.prepare` instead of being registered in a database.

After preparing, Nix writes `$digest.narinfo` next to the entry, and adds the build trace entry and metadata to the user's Trust DB.

Store objects can also come from elsewhere, for example `nix store add` or a substitution. They follow the same steps.

### Adding entries

We use rename semantics to provide atomic additions. Prepared `$cas` entries are moved to their final location with a `rename` call, which is atomic but requires the path to be on the same filesystem.

Atomicity is important to ensure that `$cas` entries are always valid. If they are copied instead, they don't self-validate for the duration of the copy.

#### with Store Daemon

Any user with write access to `/nix/store/.prepare` and `/nix/store/.stage` can ask for entries to be added. To do so:

1. They prepare entries in `/nix/store/.prepare`, each as `$cas` and `$digest.narinfo`.
1. They atomically move prepared paths to `/nix/store/.stage`, in reverse dependency order, meaning dependencies of an entry are moved first. First the `$digest.narinfo` file is moved and then the `$cas` entry.

When the Store daemon discovers a new `$cas` entry under `.stage`:

1. If the Store already contains this `$cas` entry, it removes this new one, perhaps first verifying the Store copy.
1. It recursively changes ownership of `$cas` and `$digest.narinfo` to itself and timestamps to 1, making sure that write permission is removed for everybody, and read permission is added for anybody.
   If it has no permissions to do this, it instead copies the path into `/nix/store/.daemon`, and another process will need to keep `.stage` clean.
1. The daemon verifies the `$cas`. If it doesn't match, it removes `$cas` and `$digest.narinfo`. Note that a missing or altered `$digest.narinfo` file won't pass validation.
1. It checks that all references are already present in the Store. If not, the path is held for a while and deleted if the references don't appear in time (configurable).
1. It atomically moves `$digest.narinfo` into `/nix/store`.
1. It atomically moves `$cas` into `/nix/store`.

Note that to ensure atomicity, `.prepare` and `.stage` need to be on the same filesystem, and either `.stage` or `.daemon` need to be on the same filesystem as the Store.

The daemon discovers new entries by watching `.stage`, so no communication is needed. This works from containers and from other hosts, and watching is cheap with inotify on the file server. Where inotify doesn't work, for example on an NFS client, the daemon polls instead.

Optionally, the same code is available as a helper that a user calls to process their prepared entry right away, for example as a setuid executable. This is just a fast path; without it, the daemon picks up the entry anyway.

#### without Store Daemon

Any user with write access to `/nix/store/.stage` and `/nix/store` can add entries. To do so:

1. They prepare entries in `/nix/store/.stage`, each as `$cas` and `$digest.narinfo`.
1. They atomically move prepared entries to `/nix/store`, in reverse dependency order, meaning dependencies of an entry are moved first, and `$digest.narinfo` is moved before `$cas`

Note that to ensure atomicity, `.stage` needs to be on the same filesystem as the Store.

Note that when two writers are trying to add the same `$cas` or `$digest.narinfo`, one of them might get an error, but the end result will be the same (as long as the `$cas` is self-valid). So multiple writers can also be on separate hosts, in a trusted setting.

### Verification

A path in the Store is verified by checking `$digest.narinfo` and the contents as described in In-band metadata. Note that this is more than `nix store verify` does today: it checks the contents against `NarHash` from the database, and only checks `CA` against the store path. That is fine when Nix wrote the database itself, but here anybody adding an entry writes its `$digest.narinfo`. If it doesn't match, the path is moved to `/nix/store/.quarantaine`, where a sysadmin has to investigate.

Any process with write access to `/nix/store` and `/nix/store/.quarantaine` can do this, for example the Store daemon.

### Garbage collection

Garbage collection needs to identify store paths that are not used by anything on any of the systems sharing the same store. Instead of coordinating each run between hosts, every host keeps its GC roots up to date in the Store itself, and a collector only needs to read them.

`.gc` contains:

- `hosts/$host/roots/`: the root `$cas` entries of `$host`, as 0-length files named `$cas`. The host updates them whenever its roots change, for example after switching or pruning a profile.
- `hosts/$host/temp/$build/`: the temporary roots of a running build or substitution on `$host`, in the same format.
- `hosts/$host/alive`: touched by the host periodically, for example every hour.
- `trash/`: entries that are being removed.
- `lock`: held by a running collector, containing a timestamp.

Each host follows these rules:

- Before staging an entry, or before relying on an entry that is already present, it adds that entry to its temporary roots. This also covers builds that take days, since their inputs stay rooted until the build ends.
- When the build or substitution is done, its results are part of the regular roots, and the temporary roots are removed.
- It prunes its own profiles, like `nix-collect-garbage --delete-older-than` does today, and then updates its roots.

Any host with write access can then collect garbage:

1. It creates `.gc/lock`. If the lock already exists and is recent (for example less than a day old), another collector is running and it stops.
1. It lists the Store entries.
1. It reads the roots of all hosts and calculates their closures, using the references in `$digest.narinfo`.
1. It atomically moves each listed `$cas` that is not in a closure to `.gc/trash`, together with its `$digest.narinfo`.
1. It waits for a grace period, for example an hour.
1. It reads the roots again. Trashed entries that became reachable are moved back, first `$digest.narinfo` and then `$cas`.
1. It deletes the rest of `.gc/trash`. It also removes the `$digest.narinfo` files in the Store that have no matching `$cas`, but only if the previous run also saw them without one, and they are not in any temporary roots. Note that when adding, the `$digest.narinfo` appears shortly before `$cas`, and timestamps are always 1, so their age can't be used. Neither can `ctime`: POSIX leaves it to the implementation whether `rename` updates it, and NFS doesn't specify it for the renamed object.
1. It removes `.gc/lock`.

A host that needs an entry during the grace period can move it back from `.gc/trash` itself, after adding it to its roots.

A host whose `alive` file is older than a configurable time, for example a week, is considered gone, and its directory is removed. When it comes back, it records its roots again and fetches any missing entries. Note that a host that can't reach the Store can't add entries either, so a network split only delays collection.

For a single-user installation or a non-shared Nix store, none of this is necessary, and the GC process remains unchanged.

## Profiles

Nix profiles and GC roots stay as they are, see [profiles]. User profiles already live under `$XDG_STATE_HOME/nix/profiles`, system profiles and roots under `/nix/var/nix`.

Since a profile points to an immutable `$cas` path, it is the same across systems and can therefore be part of a network-mounted home directory.

However, a profile link itself is trusted information, and should be shared between users and systems only when they trust each other.

For a shared store, the GC roots of each host are recorded as described in Garbage collection.

## Tooling

Without a store database, querying happens through the Trust DBs. Listing the Store was always a hack to query it, so the tools need to make this smooth:

- A query tool, for example `nix show tool`, lists the known entries matching a name or description from the Trust DBs, with their metadata, and highlights the ones present in the Store. Unlike `nix search`, which searches expressions, this searches what was built or is known to a source.
- `nix path-info` and `nix log` show the names from the Trust DB next to store paths.
- Each user can add, remove and order the sources of their Trust DB.
- `nix store verify`, `nix store repair` and garbage collection work on the Store as described above.

## Administration Tasks

### Migration

Nix already converts a closure to content-addressed form with `nix store make-content-addressed`. After that, each entry only needs its `$digest.narinfo`, which can be generated from the Nix Store DB.

Once garbage collection has removed the input-addressed paths, the Store only holds self-validating entries, and the Nix Store DB is no longer needed.

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

- Wait for pending additions to complete.
- Stop Store Daemon.
- Change permissions on the Store as desired.

### Repairing an entry

As Nix already does with `nix store repair`. Since `$cas` entries need no signature, any cache that has it will do.

## Implementation

This is a new store type next to the existing ones, not a replacement. Nix's local store keeps mixing input-addressed and content-addressed paths as it does today, only this Store holds content-addressed entries exclusively. Hosts can switch over one at a time: a host uses either its classic store or the Store, and `nix copy` moves closures between them.

Note that the Store could start out holding only closures converted with `nix store make-content-addressed`. However, the Trust DB needs build trace entries, so this RFC depends on content-addressed derivations being usable for Nixpkgs. They don't need to be stable first.

### Already in Nix

- Floating content-addressed derivations, with scratch paths, rewriting and resolution, behind the `ca-derivations` experimental feature.
- Calculating and verifying content-addressed store paths: `ValidPathInfo::isContentAddressed` recalculates the store path from `CA` and the references, and `checkSignatures` doesn't need signatures for content-addressed paths. Recalculating `CA` from the contents happens when adding a path (`LocalStore::addToStore`), not in `nix store verify`.
- Reading and writing `.narinfo` files. The local binary cache store (`file://`) already lays out `.narinfo` files and `build-trace-v2/` entries in a directory. The Store is close to that, but with unpacked entries instead of compressed NARs, so most of the store layer code carries over.
- The build trace, as the SQLite table `BuildTraceV3` (`drvPath`, `outputName`, `outputPath`, `signatures`). Entries are substituted from binary caches, and their signatures are checked against `trusted-public-keys`.
- Canonicalising permissions and timestamps.
- `nix store verify`, `nix store repair`, `nix store optimise`, `nix store make-content-addressed` and `nix copy`.
- Registering store types (`store-registration.hh`), and loading extra shared libraries with `plugin-files`. So the new store type can be prototyped out of tree, as a plugin.
- Building is being decoupled from the local store, behind a `BuildingStore` interface. However, registering the outputs still requires a `LocalStore`.
- The IPC builder protocol (`builder-rpc-v0`), in development.
- The local overlay store, chroot stores and the read-only local store.

Outside of Nix:

- Hydra can build content-addressed derivations, including early cutoff and dynamic derivations. Its manual still calls this "highly experimental".
- Nixpkgs has `config.contentAddressedByDefault`, which makes every derivation content-addressed. It is marked as a mass rebuild.

### To write

1. **The store type**: a `LocalFSStore` without SQLite.
   - An entry is valid when `$cas` and `$digest.narinfo` are present and validate.
   - `queryPathInfo` reads `$digest.narinfo`, and `queryPathFromHashPart` is a direct lookup.
   - `addToStore` goes through `.prepare`, `.stage` and `rename`.
   - Note that garbage collection only needs references, not referrers. Referrers can be calculated on demand and cached locally.
1. **Building into the Store**: finish decoupling the output registration from `LocalStore`, so local builds can produce entries for the new store type. This follows the direction Nix is already going.
1. **The Trust DB**: move the build trace out of the store database, into one SQLite DB per user and per source, with an order of precedence. Resolution looks up entries in the user's Trust DBs. This also needs configuration for the sources, the subjective metadata fields and the maintenance rules.
1. **The Store daemon**: watching `.stage`, with inotify on Linux, FSEvents or kqueue on macOS, and polling as a fallback. It validates, canonicalises, quarantines and adds entries. The optional setuid helper uses the same code.
1. **Garbage collection for a shared Store**: roots per host, temporary roots, `.gc/trash` with its grace period, and the lock. The existing garbage collection stays for non-shared stores.
1. **Tooling**: the query tool, names in `nix log` and `nix path-info`, and managing Trust DB sources, see Tooling.
1. **Small things**:
   - The `.narinfo` parser has to accept files without `URL`.
   - A helper that writes `$digest.narinfo` from the Nix store database, for Migration.

### Infrastructure

1. **Nixpkgs built content-addressed, at scale**: today cache.nixos.org only has input-addressed builds. Hydra needs a jobset with `contentAddressedByDefault`, that uploads NARs, `.narinfo` files and build trace entries. Until the switch-over, that is a second copy of the world.
1. **Distributing build traces**: binary caches already serve build trace entries per key. What's missing is an incremental feed, for example a JSON array of added and changed entries since some timestamp, or a downloadable SQLite file. That way, a Trust DB can be filled without one lookup per derivation. Each source also needs its own signing key.
1. **Binary caches**: nothing new for the core of this RFC. Moving the Store to `/var/lib/nix` would need yet another set of builds and caches, see Incidental improvements.
1. **Measurements**: quantify early cutoff and FUSE path masking on a Hydra-sized store, see the TODO list.
1. **Tests**: NixOS VM tests for each case study, especially multiple hosts and containers sharing a Store over NFS.

Note that this still needs someone to implement it. Prototyping the store type as a plugin lets that start outside of Nix, before anything needs signing off.

## Alternative options

There are a few choices made in this RFC, here we describe alternatives and why they were not picked.

### No metadata

Not keeping metadata in the Store means that the Store by itself doesn't have enough information to do garbage collection, nor to let a system boot from a given `$cas`.

### Keep metadata in directory with Store entry

Since the Store entries can be files or directories, that means that files would have to be put in a directory, for example `$cas` becomes `$cas/_`.
Then directory entries would have to do the same for symmetry. This requires many code changes and requires extra storage, even if an entry doesn't have any runtime dependencies.

### Store entries in subdirectories

The entries could be spread over subdirectories, for example `/nix/store/ab/$cas`, to speed up listing the Store.

However, modern filesystems handle large directories fine, and subdirectories mean more fragmentation, so listing doesn't get much faster. It also breaks every script that constructs store paths. Listing the Store is a hack to query it anyway, it's better to query the Trust DB.

## Incidental improvements

These are not needed for the rest of the RFC. However, since we're working on the store layer anyway, they are cheap to do at the same time. The first two also go well together, since dropping the name makes room for the longer store directory.

### Remove the name from store paths

Store paths become `/nix/store/$digest`, so `$cas` is just `$digest`. The digest is calculated as Nix does, but with an empty name.

The name and version are subjective data: two sources could name identical content differently. They move to the Trust DB, where the rest of the subjective metadata already is.

As a bonus, identical content deduplicates even when it was built under different names, for example `hello` and `hello-2.10`. With the name in the path, those are two entries. This happens more than you'd think: fixed-output derivations with the same hash but a different name, or copies of the same source directory with `src = ./.;`.

It also gives a single notion of equivalence: two entries are the same if their content is the same, which makes the Store easier to integrate with other content-addressed systems.

The drawback is that the store becomes more opaque and requires good tooling for manual management. Build logs and error messages only show hashes, which makes debugging harder. To remedy, `nix log` and `nix path-info` can show the names from the Trust DB, see Tooling.

### Move the Store to `/var/lib/nix`

This makes the Store FHS compliant, and makes it easier to use Nixpkgs where creating `/nix` is not possible.

An [informal discussion](https://discourse.nixos.org/t/nix-var-nix-opt-nix-usr-local-nix/7101) concluded that the Store should be located at `/var/lib/nix` for maximum compatibility.

Nix chroot stores already allow a store at another physical location, but they require mount and user namespaces and only work on Linux. Changing the logical store directory is also already possible, but loses the binary caches. Here we make `/var/lib/nix` a supported default instead, so binary caches serve it too.

Since the store directory is part of every store path digest, the same content gets a different `$cas` under `/var/lib/nix` than under `/nix/store`. Binary caches need to serve both for the transition period.

A separate location also lets a classic `/nix/store` with input-addressed paths coexist with the Store on the same host, which is convenient while migrating.

As for the contents of `/nix/var`, all of it can go elsewhere:

- `/nix/var/log` should go under central or per-user log.
- `/nix/var/nix`:
  - `db`: Store database. Objective metadata moves into the Store (see Metadata), subjective metadata into Trust DBs.
  - `daemon-socket`: Builder service. Should move to appropriate location for service sockets, like `/run`
  - `gc`: See the Garbage Collection section
  - `gcroots`, `profiles`: system profiles and roots go under `/var/lib/nix-profiles`. User profiles already live under `$XDG_STATE_HOME/nix/profiles`.
  - `temproots`, `userpool`: Builder service. Should move to appropriate locations for services, like `/var/tmp` and `/var/lib`

Nix already supports setting these per store, so this is a change of defaults.

NixPkgs needs to be audited to remove hard-coded `/nix` names, replacing it with `builtins.storeDir`.

#### Migration

To migrate an existing path `/nix/store/$old-$name` to `/var/lib/nix/$digest`, the following approach will work most of the time:

- migrate all its references using the below steps
- calculate the new `$digest` as described above, but replace all strings of the form `/nix/store/$old-$name` with `/var/lib/nix/$filler$digest`
- do the same with symlinks, but consider relative paths as well
- write `$digest.narinfo` and place the entry in `/var/lib/nix/$digest`

`/var/lib/nix/` is 2 characters longer than `/nix/store/`, but the old path also has `-$name`, which is at least 2 characters. So there is always room, and the leftover is filled with `$filler`, a string of length `l = length($name) - 1` of the form `./././/`. That is, repeat `./` `floor(l/2)` times and append `/` if `l` is odd.

Note that Nix scans for references by digest, so the filler doesn't hide any references.

### Provide the Store via FUSE

Instead of a plain directory, the Store daemon can provide the Store as a FUSE filesystem, or something similar like a network filesystem server. The entries are kept in a backing directory that only the daemon accesses.

This brings a few things:

- **Verification on access**: the daemon verifies checksums while files are read, so a corrupted entry is detected when it's used, not when a verification run happens to pass by. The NAR hash covers a whole entry, so this works best with per-file hashes, like the Git method of content addressing that Nix already has. However, that method doesn't support references yet, and only uses SHA-1 for now, so this needs SHA-256 Git hashing first.
- **Fetching on demand**: when a missing entry is accessed, the daemon fetches it from a binary cache, hanging the I/O request until it's downloaded and verified. Everything is always installed, and installing a profile only means creating the link.
- **Better deduplication**: the daemon can mask store paths in files in the backing directory, replacing them with a placeholder like Nix already does for self-references. The masked paths are kept separately, for example as a list of file, offset and reference per entry, and put back on read. Files that only differ in the store paths they contain, like a library rebuilt against a new dependency, then become identical on disk and are hard-linked via `.links`.

The backing directory could also be something else entirely, for example a [git repository that splits out references](https://gist.github.com/wmertens/eceebe0fc05461ebdc8fb106d90a6871), or casync.

The savings can be large. In informal measurements on a laptop store (by jameysharp, in the discussion of this RFC in March 2020), 52% of 3,211 store paths had duplicates when ignoring the store paths they contain, and making ELF runpaths constant would shrink 3.9GB of binaries to 2.4GB. This still needs quantifying on a Hydra-sized store.

Note that the Store as seen through the filesystem doesn't change, so `$cas`, `$digest.narinfo` and verification stay the same. The masking is purely a storage detail of the daemon.

The drawbacks are the FUSE overhead, limited FUSE support outside Linux, and that the daemon must run before anything in the Store can be used. Fetching on demand also needs network access at unexpected moments, so it should be configurable per host.

### Make the Store non-listable

The Store directory gets mode `dr-x--x--x`, so only its owner can list it. Everybody else can still access entries, but only if they know the path.

This way, nobody can casually enumerate what is installed on a host, or on a shared Store, what the other hosts have installed. The paths you need come from your own Trust DB and profiles anyway.

Note that this is not access control: digests are public in binary caches and build traces, so anyone who knows what to look for can still check if an entry is present.

Garbage collection, verification and the Store daemon need to list the Store, so they run as the owner. For single-user installs the owner is the user, so nothing changes there.

The drawback is that tab completion of store paths and `ls /nix/store/*tool*` stop working. Instead, a tool like `nix show tool` can list the known entries from the Trust DB and highlight the ones that are present.

## Future work

### Late binding

Many rebuilds only change the store paths of dependencies inside an entry. If entries referred to their dependencies by name, and a wrapper or loader configuration bound those names to store paths at runtime, more rebuilds would produce the same `$cas`. Early cutoff would then stop a lot more rebuilds, for example after a small change to openssl or bash. This also helps installs on low-power systems, since fewer entries need building or downloading.

This touches the dynamic loader, `makeWrapper`, runpaths and interpreter paths, so it needs its own RFC. Providing the Store via FUSE already gets the disk space savings, see Incidental improvements.

[RFC 62]: https://github.com/NixOS/rfcs/blob/master/rfcs/0062-content-addressed-paths.md
[building]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/building.md
[store path calculation]: https://github.com/NixOS/nix/blob/master/doc/manual/source/protocols/store-path.md
[build trace]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/build-trace.md
[resolution]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/resolution.md
[narinfo]: https://github.com/NixOS/nix/blob/master/doc/manual/source/protocols/binary-cache/narinfo.md
[local store]: https://github.com/NixOS/nix/blob/master/src/libstore/local-store.md
[local overlay store]: https://github.com/NixOS/nix/blob/master/src/libstore/local-overlay-store.md
[profiles]: https://github.com/NixOS/nix/blob/master/doc/manual/source/command-ref/files/profiles.md
