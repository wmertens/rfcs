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
- script that migrates an existing `/nix/store` closure to `/var/lib/nix`, see Incidental improvements

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
- Store can be shared read-write on a network share, with atomic installation via `rename`
- `nix-daemon` becomes optional, also for multi-user installs
- Coordinated garbage collection for shared stores
- Incidental improvements: drop the name from store paths, and move the Store to `/var/lib/nix`

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
- detect non-reproducible builds by comparing build trace entries between sources
- easily switch between single- and multi-user setup

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

- Garbage collection is more complex when the store is shared between hosts. Also, a host or user could root huge amounts of data. Quotas per host and per user are left for later.
- `$cas` entries without metadata are opaque, and might contain malware or illegal content. If nothing references it, there is no problem with the content. Garbage collection takes care of unused entries.
- A hash collision would allow inserting malware into a widely used `$cas`. This is already possible today, but trusting the hashes may lead to wider cache use. Remedies include using secure hashes, scanning for malware, using multiple hashes and comparing between binary caches, …
- Hidden self-references break content-addressed builds. When an output contains its own scratch path in a form that Nix can't find, like a compressed man page, a JAR or a signed binary, the rewrite misses it. The finished entry then points to a path that doesn't exist, and a different derivation building the same content gets a different `$cas`. This is already the case for Nix's content-addressed derivations. Such leaks are detected by building twice with different scratch paths, and fixed with rewriters in the build, for example with Nix's IPC builder protocol (`builder-rpc-v0`, in development), where the builder can unpack, rewrite and repack such files itself. References to dependencies don't have this problem, since the build already sees their final `$cas`.

Note that we don't add a fallback, like a symlink from the scratch path to the `$cas`. Such a symlink can't be validated from its contents, and it would hide the bug instead of getting it fixed.

Note that Nix already signs build trace entries against malicious mappings. This RFC doesn't change that.

### Terminology

We use the Nix concepts, with these shorthands:

- `$drv^out`: a derivation output, meaning a resolved derivation path plus output name. This is the key of a build trace entry.
- `$cas`: the base name `$digest-$name` of a content-addressed store path, as calculated by Nix (or just `$digest`, see Incidental improvements).
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

Note that the Store only holds content-addressed entries, so input-addressed paths have to be converted or removed first, see Migration.

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

Any user with write access to `/nix/store/.prepare` and `/nix/store/.stage` can ask for entries to be installed. To do so:

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

#### without Store Daemon

Any user with write access to `/nix/store/.stage` and `/nix/store` can install entries. To do so:

1. They prepare entries in `/nix/store/.stage`, each as `$cas` and `$digest.narinfo`.
1. They atomically move prepared entries to `/nix/store`, in reverse dependency order, meaning dependencies of an entry are moved first, and `$digest.narinfo` is moved before `$cas`

Note that to ensure atomicity, `.stage` needs to be on the same filesystem as the Store.

Note that when two writers are trying to install the same `$cas` or `$digest.narinfo`, one of them might get an error, but the end result will be the same (as long as the `$cas` is self-valid). So multiple writers can also be on separate hosts, in a trusted setting.

### Verification

A path in the Store is verified like `nix store verify` does for content-addressed paths, but using `$digest.narinfo` instead of the database. If it doesn't match, the path is moved to `/nix/store/.quarantaine`, where a sysadmin has to investigate.

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
1. It deletes the rest of `.gc/trash`, and removes `$digest.narinfo` files in the Store that are older than the grace period and have no matching `$cas`. Note that when installing, the `$digest.narinfo` appears shortly before `$cas`.
1. It removes `.gc/lock`.

A host that needs an entry during the grace period can move it back from `.gc/trash` itself, after adding it to its roots.

A host whose `alive` file is older than a configurable time, for example a week, is considered gone, and its directory is removed. When it comes back, it records its roots again and fetches any missing entries. Note that a host that can't reach the Store can't add entries either, so a network split only delays collection.

For a single-user installation or a non-shared Nix store, none of this is necessary, and the GC process remains unchanged.

## Profiles

Nix profiles and GC roots stay as they are, see [profiles]. User profiles already live under `$XDG_STATE_HOME/nix/profiles`, system profiles and roots under `/nix/var/nix`.

Since a profile points to an immutable `$cas` path, it is the same across systems and can therefore be part of a network-mounted home directory.

However, a profile link itself is trusted information, and should be shared between users and systems only when they trust each other.

For a shared store, the GC roots of each host are recorded as described in Garbage collection.

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

- Wait for pending installs to complete.
- Stop Store Daemon.
- Change permissions on the Store as desired.

### Repairing an entry

As Nix already does with `nix store repair`. Since `$cas` entries need no signature, any cache that has it will do.

## Implementation

- Nix needs a store type that reads objective metadata from `$digest.narinfo` instead of SQLite, and keeps the build trace in per-user Trust DBs. The tools either use the old location and semantics, or the new one.
- Binary caches already serve `.narinfo` files and build trace entries.
- Build trace entries need to be distributed in an incremental way. For example, as a JSON array of added and changed entries since some timestamp.

## Alternative options

There are a few choices made in this RFC, here we describe alternatives and why they were not picked.

### No metadata

Not keeping metadata in the Store means that the Store by itself doesn't have enough information to do garbage collection, nor to let a system boot from a given `$cas`.

### Keep metadata in directory with Store entry

Since the Store entries can be files or directories, that means that files would have to be put in a directory, for example `$cas` becomes `$cas/_`.
Then directory entries would have to do the same for symmetry. This requires many code changes and requires extra storage, even if an entry doesn't have any runtime dependencies.

## Incidental improvements

These are not needed for the rest of the RFC. However, since we're working on the store layer anyway, they are cheap to do at the same time. The first two also go well together, since dropping the name makes room for the longer store directory.

### Remove the name from store paths

Store paths become `/nix/store/$digest`, so `$cas` is just `$digest`. The digest is calculated as Nix does, but with an empty name.

The name and version are subjective data: two sources could name identical content differently. They move to the Trust DB, where the rest of the subjective metadata already is.

As a bonus, identical content deduplicates even when it was built under different names, for example `hello` and `hello-2.10`. With the name in the path, those are two entries.

The drawback is that the store becomes more opaque and requires good tooling for manual management. For example, `nix path-info` could show the names from the Trust DB.

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

NixPkgs needs to be audited to remove hard-coded `/nix` names, replacing it with the store path variable (TODO look up name).

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

- **Verification on access**: the daemon verifies checksums while files are read, so a corrupted entry is detected when it's used, not when a verification run happens to pass by. The NAR hash covers a whole entry, so this works best with per-file hashes, like the Git method of content addressing that Nix already has (it doesn't support references yet).
- **Fetching on demand**: when a missing entry is accessed, the daemon fetches it from a binary cache, hanging the I/O request until it's downloaded and verified. Everything is always installed, and installing a profile only means creating the link.
- **Better deduplication**: the daemon can mask store paths in files in the backing directory, replacing them with a placeholder like Nix already does for self-references. The masked paths are kept separately, for example as a list of file, offset and reference per entry, and put back on read. Files that only differ in the store paths they contain, like a library rebuilt against a new dependency, then become identical on disk and are hard-linked via `.links`.

Note that the Store as seen through the filesystem doesn't change, so `$cas`, `$digest.narinfo` and verification stay the same. The masking is purely a storage detail of the daemon.

The drawbacks are the FUSE overhead, limited FUSE support outside Linux, and that the daemon must run before anything in the Store can be used. Fetching on demand also needs network access at unexpected moments, so it should be configurable per host.

### Make the Store non-listable

The Store directory gets mode `dr-x--x--x`, so only its owner can list it. Everybody else can still access entries, but only if they know the path.

This way, nobody can casually enumerate what is installed on a host, or on a shared Store, what the other hosts have installed. The paths you need come from your own Trust DB and profiles anyway.

Note that this is not access control: digests are public in binary caches and build traces, so anyone who knows what to look for can still check if an entry is present.

Garbage collection, verification and the Store daemon need to list the Store, so they run as the owner. For single-user installs the owner is the user, so nothing changes there.

The drawback is that tab completion of store paths and `ls /nix/store/*tool*` stop working. Instead, a tool like `nix show tool` can list the known entries from the Trust DB and highlight the ones that are present.

[RFC 62]: https://github.com/NixOS/rfcs/blob/master/rfcs/0062-content-addressed-paths.md
[building]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/building.md
[store path calculation]: https://github.com/NixOS/nix/blob/master/doc/manual/source/protocols/store-path.md
[build trace]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/build-trace.md
[resolution]: https://github.com/NixOS/nix/blob/master/doc/manual/source/store/resolution.md
[narinfo]: https://github.com/NixOS/nix/blob/master/doc/manual/source/protocols/binary-cache/narinfo.md
[local store]: https://github.com/NixOS/nix/blob/master/src/libstore/local-store.md
[local overlay store]: https://github.com/NixOS/nix/blob/master/src/libstore/local-overlay-store.md
[profiles]: https://github.com/NixOS/nix/blob/master/doc/manual/source/command-ref/files/profiles.md
