- Feature Name: retained_filesystem
- Start Date: 2026-10-21
- RFC PR: [rfcs#41](https://github.com/astrid-runtime/rfcs/pull/41)
- Tracking Issue: [rfcs#40](https://github.com/astrid-runtime/rfcs/issues/40)
- Status: Draft; no ABI registration, SDK publication or provider activation

# Summary

Add an opt-in retained filesystem interface in a new `astrid:fs@1.1.0`
package. It exposes identity-bound regular files and directories, explicit
permission availability, atomic creation and namespace operations, and bounded
snapshot enumeration. Keep `astrid:fs@1.0.0` byte-for-byte unchanged. Existing
capability, principal, export and lifecycle authority remains the enforcement
boundary; a node identifier is descriptive and conveys no authority.

# Motivation

The Linux Realm currently implements its home filesystem inside the capsule.
Moving that home into the canonical kernel-owned store must preserve the actual
filesystem contract, including retained files after unlink, directory handles
after rename, executable and zero permission bits, exclusive creation and
restart durability. A path-based adapter with best-effort mode reporting cannot
preserve these properties. In particular, creating a file and reopening its
name can return a concurrent replacement; caching a directory pathname can
redirect an existing handle after rename and recreation.

This proposal supplies the host boundary for that migration. It does not itself
migrate a home, authorize an owner, stop a guest or permit concurrent old and new
writers. Those prerequisites must be established independently before activation.
Hosted implementations remain portable. Standalone implementations may implement
the same interface without inheriting a hosted OS trust claim.

# Guide-level explanation

A capsule explicitly imports `astrid:fs/retained@1.1.0`. It opens an authorized
export root, then selects children through retained directory resources. Lookup
returns a selected-node resource before a file access mode is chosen. Each child
name is one component, not a VFS path. File creation returns the selected file
resource directly. No second pathname lookup is required.

For example, a caller opens `home://`, selects `agent`, then exclusively creates
`report` with permission bits `0o600`. If another caller wins creation, the
operation returns `already-exists`. On success, the returned file still denotes
that original object if another caller renames or unlinks `report` and creates
a replacement. Reads, writes, stat and permission changes use the retained
identity. Dropping a handle releases custody; it does not unlink a name.

A retained directory follows its original directory after a rename within the
admitted export. Recreating its old name never redirects it. If the original
object is removed or leaves the admitted export, child operations fail explicitly.
A directory capability cannot follow an ancestor outside its original export.

Permission bits are either unavailable or an explicit integer in `0..=0o7777`.
`Some(0)` means mode `0000`; it never means unavailable. Persisting these bits
is not a claim that the host implements POSIX user/group discretionary access
control. Astrid's authenticated principal and capability gates still apply.

# Reference-level explanation

## Version and interface coexistence

Create a new versioned WIT file. Its `host` interface retains the existing
1.0 host declarations without shape changes; the additive `retained` interface
contains the new operations. The kernel registers explicitly supported interfaces
and versions. The old package remains registered independently. A capsule opts
into the new import rather than acquiring it through an SDK default change.

Resources from different interfaces or versions are nominally distinct. No
resource cast, integer-handle conversion or implicit version negotiation is
provided. Internal storage, quota and custody mechanisms should be shared.
This RFC requires a separately reviewed WIT definition before implementation;
the operation names below specify the proposed semantic surface.

## Facts and errors

`node-stat` contains:

| Field | Meaning |
| --- | --- |
| `identity` | Opaque fixed-size logical identity, separated by export incarnation. Neither pathname, content digest, native inode nor authority token. |
| `revision` | Opaque object revision comparable for equality; real metadata/content changes replace it. No monotonic arithmetic or global ordering is promised. |
| `kind` | Regular file or directory for this interface. |
| `size` | Logical byte length for a regular file; zero for a directory, not aggregate descendant size. |
| `permissions` | Optional POSIX permission bits; values outside `0o7777` are rejected. |

The interface does not synthesize timestamps, UID/GID, hard links, symlinks or
device nodes. Unsupported backend features return `unsupported` before mutation.
It must not report invented permissions or identities to claim conformance.

Errors include the applicable existing typed filesystem errors, plus `stale`
(original selected lifetime or directory no longer usable), `conflict`
(captured-root publication lost to another writer), `unsupported` and
`invalid-options`. Errors contain no host paths, secrets or raw backend dumps.
`not-found` describes a missing child of a valid directory; it does not substitute
for a retired directory. A resource with a retired owner/export lifetime cannot
recapture current authority by retrying the same text identifier.

## Export and directory resources

`open-root(path, access)` admits a VFS export root through the existing authenticated
mount/capability mechanism and returns a directory resource and its exact stat.
Only a complete authorized root URI is accepted here; it is not an arbitrary
catalog-root selector. The implementation captures immutable owner identity,
export identity and generation, selected directory identity and lifecycle
custody. The export root is a real typed directory with persisted metadata.
Directory access is `read` or `read-write`; child directory selection explicitly
requests a subset of its parent resource rights. A file resource captures the
requested file access. Neither moving a resource nor supplying another
principal in payload data can change that captured authority.

Every resource call must originate from the captured authenticated principal
and revalidate current capabilities, mount policy and original lifetime.
Read/stat/enumeration require read authority; byte writes, resize and chmod
require write access on the selected resource and current write capability.
Creation, unlink, rmdir and rename require a write-enabled directory and current
write authority; rename requires it on both directory contexts. Metadata
availability does not supply an alternative permission grant. Revoked or
cross-principal resources fail even while their underlying object remains live.

Directory operations are `stat`, `set-permissions`, `open-directory(name)`,
`open-file(name, options)`, `create-directory(name, permissions)`, `unlink(name)`,
`remove-directory(name)`, `rename(name, destination-directory, new-name)` and
`enumerate`. Directory creation returns its exact created directory resource.
The interface has no recursive delete, arbitrary parent traversal or path
canonicalization operation that can enlarge authority. The bounded parent
selection below remains inside the original admitted export.

A child name is nonempty UTF-8 satisfying existing canonical path validation,
with no slash, NUL, control character, `.` or `..`. Name limits are checked before
backend allocation and again at the authoritative resolution boundary. There
is no normalization that silently aliases two distinct input names.

Every operation validates the captured original lifetime and export boundary.
Resolution of a renamed directory derives from the authoritative catalog at
one captured root. A cache is valid only for that root and cannot replace
identity validation. Unlinked directory resources may retain metadata custody,
but cannot create hidden descendants or access a replacement name. Movement
outside the captured export makes child operations fail `boundary-escape` or
`stale`; returning to a path does not revive terminally retired authority.

## Selected nodes before file opening

`directory.lookup(name)` returns a selected-node resource and its exact stat.
Lookup captures the original logical identity and owner/export lifetime, and
registers custody in the same authoritative root observation that selects the
child. It does not grant byte read/write access, create or truncate anything.
The original export and directory rights constrain every later operation.

`directory.parent(access)` selects the authoritative current parent within the
same original export, with current policy checks, rights narrowing and resource
custody admission. At the captured export root it selects that same root. After
a directory rename it uses the current authoritative parent, not a remembered
walk stack or a former textual path. An unlinked or out-of-export directory
returns `stale`. It cannot reach an owner namespace or native parent outside the
export, and it does not provide arbitrary `..` path canonicalization. This
supports a 9P parent walk without increasing authority.

A selected node exposes `stat`, `clone`, directory-only `lookup(name)` and
`parent`, `open-file(access, truncate)` and `open-directory(access)`. Selected-node
traversal admits current read policy while preserving the original export's
access ceiling for a later explicit file open; traversal itself grants no byte
IO. In contrast, an explicitly narrowed opened-directory resource constrains the
ceiling of every descendant it returns. Opening a file for writing must neither
require write access merely to traverse nor recover rights removed by an
explicitly narrowed directory. These operations revalidate the captured authenticated
principal, current policy and original export lifetime. File opening uses the
exact selected identity; it never resolves the node's former pathname. Requested
access is explicit, including write-only, and cannot exceed the original export
rights. Lookup need not request write access merely because a later file open
might do so. Truncation requires freshly admitted write access and occurs only
after return-resource and mutation admission. A node of the wrong kind returns
the corresponding typed kind error before effects.

Cloning retains the same selected object and original authority. It reserves a
new selected-resource slot before changing a caller's resource table; failure
leaves the original resource and namespace unchanged. This supports a zero-name
9P walk without a new pathname lookup. It does not duplicate an access grant or
replace current policy checks. A descriptive identity or QID cannot be converted
into a selected-node resource.

An unlinked regular node may still be opened as that original object while its
custody and original authority remain valid. Recreating its former name does
not redirect it. An unlinked directory may retain stat custody but cannot resolve
children or create hidden descendants. A renamed directory remains usable only
inside its captured export, subject to its original lifetime and current policy.

Selected-node removal, where supported, is identity-conditional: it removes a
binding only if that binding still denotes the captured node in the captured
directory context. A changed or absent binding returns an explicit stale result;
it never removes a replacement at the old name. Directory `unlink(name)` remains
a current-name operation relative to the retained parent. Neither operation
recovers authority by replaying a former path.

Selected-node, directory, enumeration-cursor and opened-file resource admission
must be finite and explicit. Their ledgers may be shared internally, but the
existing 16 opened-file ceiling and 1,048,576-byte positional IO ceiling do not
bound unopened selections or directory cursors. No new numerical default is
established by this RFC. Every returned resource, including clone and lookup,
requires its resource-table and custody admission before publication or effects.

## Atomic file opening and creation

Open options specify access (`read`, `write`, `read-write`), disposition
(`existing`, `open-or-create`, `exclusive-create`), `truncate`, and optional
creation permission bits. There is no implicit append cursor: the interface
uses positional IO. Truncate requires write access; creation requires explicit
permissions; creation permissions on `existing` are invalid. Read-only creation
and truncation are rejected. These combinations are validated before mutation.

One authoritative root attempt selects or creates the regular file and checks
parent identity/type and destination predicates. Before guarded publication it
reserves the return resource and registers custody of the exact selected object.
It then commits metadata and any truncation and returns that exact file plus
stat. There is no publication-to-custody gap in which unlink or reclamation can
consume the newly selected object. Existing
files retain their permissions. Exclusive creation never replaces an object.
An existing directory returns `is-directory`. Implicit missing-parent creation
is not allowed.

Resource capacity, metadata admission and output capacity are reserved before
publishing creation or truncation. A resource-table insertion failure must not
leave a newly created file or a truncation behind. Backend failures before
publication leave no visible mutation. If the caller disappears after a durable
publication, the operation may have committed; reconciling a lost response uses
the ordinary authoritative namespace, not an undocumented automatic retry.

## Retained regular files

Methods are positional `read-at`, `write-at`, `set-len`, `sync-data`, `sync-all`,
`stat` and `set-permissions`. They operate on the captured object, including
after unlink or replacement of its former name. They never reopen by pathname.

The per-call positional IO ceiling is 1,048,576 bytes, enforced before guest
input lifting and before output allocation. Offset/length addition is checked.
Reads return up to the admitted amount, with empty output at EOF. Writes report
the committed byte count; sparse extension is zero-filled. Changing permissions
does not change identity, bytes or link state. A no-op metadata change preserves
revision; a real committed change replaces it. Backend-specific revision wrap
must not silently equate a changed object with a stale cached revision.

Successful `sync-all` establishes backend-documented data and metadata durability;
`sync-data` may omit metadata only when the backend can uphold that distinction.
An implementation with an atomic durable root may provide the stronger behavior
for both. An unsupported durability operation must fail rather than acknowledge
volatile state. No guarantee against destruction of the underlying host storage
or rollback of an entire hosted image is implied.

## Namespace linearization

All predicates and updates for create, mkdir, unlink, rmdir and rename use one
authoritative owner root and one guarded publication. A stale root returns
`conflict` without mutating the newer root. Implementations may internally retry
only within a finite shared operation-work/retry budget, revalidating the
original resource lifetimes and all predicates on each attempt; they
cannot replay a consumed stream, redirect to a replacement directory, or report
success for a different selected file. A response describes the winning attempt.

`unlink` removes a regular-file name only. `remove-directory` removes an empty
directory only; a concurrent child cannot be lost. Root removal and root movement
are refused. Rename preserves source identity, permissions and complete subtree;
replacement is permitted only for compatible node kinds, with an empty directory
destination. Replaced objects retain existing file custody. Moving a directory
into itself or a descendant is refused. Same-name/same-directory rename is an
identity-checked no-op. Cross-export rename returns `cross-vfs` and changes neither
side. Same-export two-directory rename is atomic; two separate mutations are
not conforming.

Every supported writer to the canonical namespace must use compatible guarded
semantics. A correct 9P adapter cannot compensate for another host API performing
stale check-then-write mutations into the same namespace.

## Snapshot enumeration and bounded resources

`enumerate` creates an opaque snapshot cursor bound to the selected directory,
original export lifetime and one captured catalog root. `next` accepts bounded
entry and encoded-byte budgets and returns entries with name and `node-stat`,
plus explicit exhaustion. All pages describe that snapshot, including after
concurrent rename or replacement. Retirement invalidates further use; it never
rebinds to a new lifetime. The cursor owns necessary canonical metadata custody
through backend compaction. Dropping it releases that custody without IO.

Once admitted, an immutable enumeration may finish after directory unlink or
movement: it describes historical selected metadata, not a new child lookup.
Current read authority and the original owner/export lifetime must still be
valid. Owner/export retirement invalidates the cursor terminally; ordinary
directory unlink does not rebind or invalidate the captured snapshot. A zero
entry budget, zero byte budget or budget too small for the next complete entry
returns `invalid-options` or `too-large` without consuming that entry. A failed
page leaves position unchanged unless it explicitly returns terminal `stale`;
no success page may silently omit an entry. Reserve output admission before
advancing position. A caller that loses an already-completed page response may
have advanced the cursor; this is not an exactly-once delivery protocol.

Input bytes, results, cursor state, directory resolution work, retained records
and handles require finite pre-admission. Budget failure is explicit `quota` or
`too-large`; it cannot silently skip entries or mix a new root. Flat-catalog
scanning must be metered, with restart/reconstruction covered by the same bounds.
An implementation that cannot enumerate a supported workload within its admitted
budget fails activation tests; an unbounded scan is not an acceptable fallback.

New and old interfaces share canonical quota accounting; importing both cannot
double a capsule's file allowance. Preserve the existing 16-file ceiling unless
a separately justified policy change is adopted. Directory contexts and cursors
have finite separately admitted pools and participate in aggregate runtime
resource accounting. A 9P FID clone may share its selected resource rather than
allocating another native file. The 1,024-FID protocol limit is not permission to
raise host handle capacity implicitly.

Logical occupancy, cumulative admitted writes and resident temporary memory are
separate accounting dimensions. Occupancy refunds do not refund cumulative write
work. A cumulative-write guarantee requires an actual shared ledger and charging
at the mutation boundary, including partial writes and truncation/extension;
this RFC does not treat the current occupancy check as that ledger. Exact pool
and work-budget policies must be documented and tested before activation rather
than introduced as guest-controlled knobs.

## Lifecycle and migration

An active-work lease remains owned by actual backend work through settlement,
including cancelled waiters and output lowering. Native workers retain their
own custody when an async caller goes away. Zero callers, an empty guest resource
table or a successful job reply is insufficient proof of native thread exit.

The home port requires one coordinated activation: freeze admissions to the exact
old provider generation, drain actual controller effects and native execution,
validate bounded source inventory, stage typed objects against one original root,
and atomically publish the canonical graph plus the exact legacy-head fence.
Restart/load must refuse the old writer after that fence. The old Realm format
has no persisted logical file/directory identities: allocate distinct canonical
identities for the captured source entries, retain that mapping within the one
staged migration, and start a new export/FID epoch at activation. Do not claim
continuity with old session/path-allocated QIDs. Preserve all bytes, permission
bits, explicit directories and root permissions, unrelated namespaces and
completed branch receipts. After activation, canonical identities survive rename
and are never reassigned to a replacement. No fallback to the old
RealmFs writer or KV-side shadow permission/identity map is permitted afterward.

Both 9P and canonical Realm commands must use the new authority. 9P creation uses
the exact returned resource; directory FIDs retain directory identity. QIDs are
allocated with checked per-export identity mapping, not truncated hashes or
pathname interning. Revision-to-32-bit QID conversion needs explicit wrap/cache
invalidation behavior; permanent saturation cannot hide subsequent changes.

# Drawbacks

Retained directories and snapshots require more authority and resource bookkeeping
than paths. Multi-version host registration and explicit SDK opt-in add release
coordination. Bounded cold resolution in the current flat catalog may require an
index derived from the authoritative root. These costs are necessary for the
promised semantics; a second independent directory database is not a shortcut.

# Rationale and alternatives

Path-plus-stat detects some substitutions but does not bind subsequent operations.
Reusing the workspace adapter loses zero modes, atomic exclusive creation and
retained directory behavior. Keeping all metadata in the capsule preserves a
second authority and its reclamation burden. Editing fs1.0 breaks existing
Component Model imports. This proposal instead extends the audited owner store
and retains explicit ABI coexistence.

# Prior art

The existing Astrid fs1.0 positional file resource supplies the IO pattern; the
canonical store supplies retained object identity and guarded publication. Linux
Realm's current home supplies the preservation baseline. Familiar file-descriptor
semantics motivate retained names and atomic creation, without claiming complete
POSIX compatibility or adding unsupported multi-user discretionary policy.

# Validation and unresolved questions

Before implementation approval, review the exact WIT shapes, identity/revision
encoding, bounded directory-resolution design and pool accounting. Before home
activation, all of the following require actual integrated evidence:

- Existing fs1.0 capsules instantiate unchanged; new imports remain opt-in.
- Mode zero, special bits, root0700 and retained chmod survive rename/unlink/restart.
- Capacity refusal precedes effects; competing exclusive creators have one winner.
- Parent replacement, concurrent rmdir/child creation, rename replacement/cycles and
  cross-export attempts cannot lose or redirect objects.
- An old directory FID follows its renamed original and never a recreated pathname.
- Enumeration retains one snapshot through mutation, compaction and bounded paging.
- Cancelled waiters retain active backend custody; owner/export retirement refuses
  late operations and cannot revive through alias reuse.
- Actual migrated packaged home and canonical commands pass ordinary shell, build,
  editor/temp-file, retained-file, two-client, restart and crash-recovery journeys.
- Old packages cannot resume writing after the durable migration fence.

The canonical owner representation, bounded resolver and revision-to-QID policy
must be selected through implementation evidence before claiming full directory
parity. Native platform validation and standalone verified boot remain separate.
No hard-link, symlink, UID/GID, ACL or timestamp extension is proposed here.

# Future possibilities

Additional backend kinds and richer discretionary policy can be proposed after
the retained home contract is implemented and measured. They must not silently
change the meaning of the frozen interfaces or weaken original-lifetime authority.
