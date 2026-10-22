- Feature Name: `compute_groups`
- Start Date: 2026-10-22
- RFC PR: N/A (draft)
- Tracking Issue: N/A (to be assigned)
- Related contract: [astrid-runtime/wit#28](https://github.com/astrid-runtime/wit/pull/28)
- Status: Draft; no published compute ABI or installed Linux Realm claim

# Summary

Give a capsule an Astrid-mediated way to run its own signed core-WebAssembly
worker code in a principal-scoped compute group. The capsule's Component uses
the `astrid:compute/host@1.0.0` control plane; workers use one shared memory
and a narrowly enumerated raw core-Wasm ABI. Astrid authenticates the package
and owner, admits finite resources, schedules and cancels work, and records
accounting. The host does not implement the worker's Linux, inference, or other
application algorithm.

This proposal revises the older unmerged `feat/generic-compute-groups` draft.
The WIT declarations have the same shape as that branch and the experimental
Rust SDK at `1a06329b`, but their security and resource semantics must be
updated before a public contract is claimed.

# Motivation

A WebAssembly Component cannot hand a guest closure to a host thread. The Linux
Realm needs concurrent virtual CPUs over shared guest RAM, and other capsules
may need parallel algorithms. A Linux-specific host syscall would enlarge the
kernel's algorithmic authority. An unrestricted native process or generic
module-load syscall would bypass the capsule sandbox. Compute groups provide a
small, auditable execution boundary while retaining hosted portability and a
path to a standalone scheduler.

The key property is authorization of *these bytes for this owner and this
activation*, not merely successful Wasm type-checking. A manifest declaration,
a reusable principal name, or a worker digest by itself cannot grant execution.

# Guide-level explanation

A capsule package contains its ordinary executable Component and one or more
`[[component]] type = "compute-worker"` core-Wasm modules. Each worker may
declare an ordered list of immutable assets. The package builder includes and
hashes the exact bytes. The authenticated package is published to a principal,
then the kernel creates an activation provision bound to the package generation,
immutable principal UID, capsule identity, worker IDs/digests and ordered asset
IDs/digests. The capsule never constructs that provision from manifest input.

During an authenticated invocation, the controller imports
`astrid:compute/host@1.0.0`, opens a group for one declared worker, writes
setup bytes into shared memory, and submits bounded descriptors. A deterministic
group uses one worker and FIFO dispatch; a parallel group uses the admitted
worker count. The controller observes jobs through status, poll, join and cancel.
Dropping a job observer does not cancel the job; closing its group does.

An installation with no compute policy or no authenticated principal package
cannot open a group. A worker has no filesystem, network, process, IPC or
driver authority through this ABI. It may read only the admitted immutable
asset snapshots by index. Effects beyond its memory and result code must flow
through the controller's separately authorized capsule interfaces.

# Reference-level explanation

## Control-plane contract and version

The proposed source of truth is `wit/host/compute@1.0.0.wit` in WIT PR #28,
package `astrid:compute@1.0.0`. It depends on the existing
`astrid:io/poll@1.0.0` pollable resource. The file defines request, descriptor,
status, accounting and error types; `open`; and `compute-group` and `job`
resources. This RFC constrains their behavior without adding a WIT field or
function. The declaration tokens match the older draft and pinned SDK. Once a
version reaches WIT `main`, structural changes require a new package version
with explicit linker coexistence. There is no implicit version negotiation.

`open` requires a positive initial memory page count. A zero maximum asks the
host to resolve a finite maximum against the worker declaration and remaining
effective capacity; a nonzero maximum must be at least the initial count.
`exact(0)` and `at-most(0)` are invalid. Deterministic mode admits one worker
regardless of a positive requested count. `exact(n)` in parallel mode admits
all `n` or fails; `auto` and `at-most(n)` can resolve downward, never to zero.
The resolved count and maximum are observable in `group-info`.

Descriptors are checked for overflow and containment in current shared memory.
Their offset is at least 64, protecting the ABI header. Control-plane memory
copy calls are limited to 1 MiB each. Submitted jobs are nonblocking and have
typed terminal results. Diagnostic strings are not authorization inputs.

## Publication and activation authority

The builder rejects missing, symlinked, escaping, duplicate or hash-mismatched
worker and asset paths. Admission retains verified bytes; later reads use that
snapshot rather than reopening a mutable pathname. Worker identity includes
the component ID and digest; each asset is bound by ordered ID and digest.

The kernel provides a provision only from an admitted durable package owned by
the current immutable principal UID. It independently resolves the current
publication generation and confirms it again at activation. The provision is
bound to the capsule identity and cannot be deserialized from a manifest. A
principal alias that is deleted and reused with a different UID does not inherit
the old provision. System residency and unbound/legacy package activation do
not gain compute authority. Any mismatched owner, capsule, generation, worker
or ordered asset denies activation/open without substituting newer bytes.

Package-generation binding is an admission check, not a claim that a read-only
snapshot alone prevents concurrent replacement. Publication/retirement must
fence the exact activation generation and its native workers before deleting
or replacing it.

## Raw worker ABI 1

A worker is a core-Wasm module, not a Component. It imports exactly one
`astrid_compute.memory`: shared memory32 with a declared maximum. It exports
`astrid_compute_abi_version: () -> i32` returning `1` and
`astrid_compute_run: (i32, i64, i64, i64) -> i32`. Other exports may be
present, but only the known import allowlist below is admitted. Malformed,
duplicate, incorrectly typed and foreign-module imports fail before execution.

The first 64 shared-memory bytes form the host-owned control header: magic
`ASC1` at 0, version 1 at 4, atomic cancellation generation at 8, atomic
group state at 12, resolved worker count at 16 and current memory bytes at 24;
bytes 32–63 are reserved. Worker descriptors start after that header. The
host uses an atomically observed cancellation signal and an interrupt mechanism
to stop noncooperative work.

The first public ABI 1 permits these optional imports in addition
to memory, with exactly the indicated signatures:

| Import | Signature | Meaning |
| --- | --- | --- |
| `fuel_remaining` | `() -> i64` | Remaining fuel for the active job; `-1` when inactive/unavailable. |
| `asset_count` | `() -> i32` | Number of ordered admitted immutable assets. |
| `asset_size` | `(i32) -> i64` | Byte length by index; negative on invalid index. |
| `asset_read` | `(i32, i64, i64, i64) -> i32` | Copy from asset index/source offset into shared-memory destination/length. |

`asset_read` is callable only during a job, cannot overwrite the reserved
header, caps one call at 64 KiB, checks signed conversions and overflow, and
charges fuel before copying. It returns 0 on success, or -1 invalid index, -2
invalid range, -3 excessive length, -4 insufficient fuel, -5 inactive phase.
The host may reject any worker with an unsupported import even if its ABI
version function returns 1. Assets provide input bytes, not ambient host IO.

The original unmerged draft said workers import no functions. No compute host
ABI has shipped from canonical `main`; that experimental host is not a
supported compatibility target. The first supported ABI-1 host must implement
all four optional imports, while accepting workers that use none of them. An
installer must reject a package requiring compute on a runtime without this
host contract before publication, with a useful upgrade diagnostic. This
first-public decision requires a release-inventory check before merge; if an
ABI-1 compute host has actually been published, asset-enabled workers need a
new raw ABI number and a coexistence test instead.

## Resource bounds, cancellation and settlement

Admission intersects finite process-wide worker, shared-memory, artifact,
archive-parser, table and job-fuel limits with principal policy and current
reservations. Every opened group charges its principal and the shared host
pool. Omitted descriptor fuel resolves to a finite share from effective
principal fuel-rate and job policy, narrowed to unspent capacity; an explicit
limit cannot widen policy. Zero remaining capacity denies admission. A read
import also consumes job fuel. The implementation may impose stricter bounds,
but must report denial rather than silently treat omitted fuel as unlimited.

Group cancellation rejects new jobs and signals queued and running work. A
trap or forced interruption closes the group rather than silently restarting a
stateful worker. Principal removal, capsule unload/upgrade and daemon shutdown
must retire the captured generation, including an open still building and
partially started native workers. Reservations and retained bytes are released
only after the exact native workers settle; returning a guest error, dropping a
resource, or abandoning a waiting caller is not settlement evidence. A bounded
wait may report Pending and retain a retryable retirement ticket. This rule
prevents charging another principal for an unjoined worker and prevents a
replacement generation from inheriting old custody.

Audit records should name the authenticated principal UID, capsule, package
generation, worker ID/digest, admitted bounds, submit/cancel/terminal outcome
and reason. Asset contents and descriptor bytes stay out of audit records.

## Hosted and standalone implementations

Hosted Astrid relies on the host OS, sandbox configuration, storage and Wasm
engine for process isolation, scheduling and persistence. The same WIT and
worker ABI can be implemented by a standalone Astrid scheduler, but this RFC
does not make hosted deployment a verified-boot or formally verified kernel.
Neither implementation may silently substitute native host processes for
admitted worker modules. The authority and accounting semantics must remain
observable across both deployments.

## Conformance evidence before activation

Tests must include: exact package and generation binding; alias reuse with a
new principal UID; worker/asset reorder or hash mismatch; path substitution;
the wrong capsule and system residency; old no-asset and new asset-enabled
workers; malformed imports and ABI versions; memory/descriptor overflow;
cross-principal resource isolation; finite omitted fuel; cancellation during
startup and after submission; trap and forced interruption; and retained
reservation until exact native worker join. A packaged Linux Realm must then
install this contract, open its group, boot its real guest and exercise
shutdown/restart. Source tests and a synthetic compatibility fixture do not
substitute for that installed journey.

# Drawbacks

Shared-memory workers and native execution threads enlarge the runtime's
concurrency surface. The host must account for compilation memory and startup
latency, and a slow or wedged worker can delay final settlement. The package
format and activation path become more complex. A generic control plane also
leaves algorithm-specific correctness to capsules and their own evidence.

# Rationale and alternatives

A capsule-specific Linux vCPU host call would move guest semantics into the
kernel. An unrestricted module loader would erase the signed object boundary.
Host processes would inherit a larger OS attack surface and different authority
semantics. A deterministic one-worker mode remains useful for reproduction,
but cannot supply actual parallel vCPUs. The proposed groups preserve narrow
host custody with an application-neutral data plane.

# Prior art

The older unmerged compute-groups RFC and WIT feature branches establish the
control-plane shape and 64-byte worker header. This draft keeps those shapes
while replacing the earlier assumptions that a worker imports no functions,
that omitted fuel may be unbounded, and that a manifest declaration alone is
the effective grant. WebAssembly's shared memory and fuel/interrupt facilities
provide mechanisms; they do not authenticate a principal's package on their
own.

# Unresolved questions

- Complete the release-inventory check that no supported ABI-1 compute host
  shipped before this contract; otherwise revise the raw ABI version decision.
- Specify the installer diagnostic for a package requiring compute on a runtime
  with no compute host implementation.
- Pin the exact public operator-policy schema and audit event format. The
  current candidate's finite internal admission is not evidence that every
  deployment has configured those limits.
- Decide whether a later worker ABI needs an explicit package feature set in
  addition to its `astrid_compute_abi_version` export.

# Future possibilities

Durable named groups and reattachment after daemon restart would require a
separate identity and recovery protocol. Device/GPU work belongs behind a
separate audited authority. Formal reasoning about scheduling or worker
results must be stated separately from the authorization and package evidence
specified here.
