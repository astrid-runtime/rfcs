- Feature Name: `compute_groups`
- Start Date: 2026-10-22
- RFC PR: [rfcs#42](https://github.com/astrid-runtime/rfcs/pull/42)
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
Rust SDK at `1a06329b`, but their security and resource semantics differ.
Astrid `main` and stable v2026.10.1 contain no compute host: the earlier
experimental worker ABI has no supported public compatibility promise.

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
hashes the exact bytes. Before publication, the installer checks that this
host supports the declared compute contract; otherwise it reports
`unsupported-compute-host` and publishes nothing. On a capable host, the
authenticated package is published to a principal. The kernel then
independently reads the live registry and creates an activation
provision bound to its exact generation, immutable principal UID, capsule
identity, worker IDs/BLAKE3 digests and ordered asset IDs/BLAKE3 digests. The
capsule never constructs that provision from manifest input. The host rechecks
that exact live publication at activation and group open.

During an authenticated invocation, the controller imports
`astrid:compute/host@1.0.0`, opens a group for one declared worker, writes
setup bytes into shared memory, and submits bounded descriptors. A deterministic
group uses one worker and FIFO dispatch; a parallel group uses the admitted
worker count. The controller observes jobs through status, poll, join and cancel.
Dropping a job observer does not cancel the job; closing its group does.

An installation with no finite compute policy or no authenticated principal
package cannot open a group. A worker has no filesystem, network, process,
IPC or driver authority through this ABI. It may read only the admitted immutable
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
`some(0)` job fuel is invalid. A targeted `worker-index` has occupancy one:
if it already has pending or running work, `submit` returns `busy`. Untargeted
dispatch selects a worker with the fewest pending jobs; a full bounded queue
returns `busy`. The candidate permits 64 waiting commands per worker.

Descriptors are checked for overflow and containment in current shared memory.
Their offset is at least 64, protecting the ABI header. Control-plane memory
copy calls are limited to 1 MiB each. Submitted jobs are nonblocking and have
typed terminal results. Diagnostic strings are not authorization inputs.

`grow(delta)` cannot exceed the maximum reserved at open. It returns the old
page count and updates current-memory accounting and the advisory byte-count
header; a closed group rejects growth. Cancellation rejects submit/grow but a
retained group resource may still allow bounded read/write of its shared memory
until retirement releases that allocation. Dropping the resource starts
retirement; it is not proof the last native worker has exited. A read of RAM
after cancel is not a new execution grant or an authenticated result.

## Publication and activation authority

Three distinct gates apply. Installer **preflight** checks that the target
runtime implements the compute host package and raw worker ABI required by the
signed package, before writing any publication state; failure is the named
`unsupported-compute-host` diagnostic with an upgrade hint. **Publication**
authenticates and durably binds the exact package bytes and generation to an
immutable principal UID. **Activation and each `open`** independently recheck
that exact live owner/capsule/generation under the registry-retirement gate.
Passing installer preflight does not prove publication or grant a later open.

The builder rejects missing, symlinked, escaping, duplicate or hash-mismatched
worker and asset paths. Admission retains verified bytes; later reads use that
snapshot rather than reopening a mutable pathname. Worker identity includes
the component ID and BLAKE3 digest; each asset is bound by ordered ID and BLAKE3
digest. A worker has at most 16 ordered assets.

The kernel provides a provision only from an admitted durable package owned by
the current immutable principal UID. It captures the exact live registry
generation when constructing that provision, then rechecks the same
owner/capsule/generation under the publication-retirement gate at activation
and every group `open`. The provision is bound to capsule identity and cannot
be deserialized from a manifest. A
principal alias that is deleted and reused with a different UID does not inherit
the old provision. System residency and unbound/legacy package activation do
not gain compute authority. Any mismatched owner, capsule, generation, worker
or ordered asset denies activation/open without substituting newer bytes.
Revocation racing an already admitted open cancels
that exact generation; it cannot silently complete under a replacement.

Package-generation binding is an admission check, not a claim that a read-only
snapshot alone prevents concurrent replacement. Publication/retirement must
fence the exact activation generation and its native workers before deleting
or replacing it. An unlocked live-registry observation is insufficient; if
the implementation cannot hold or revalidate that gate, it must fail closed.
This is a normative requirement, not proof the current candidate has closed
every publication race.

`open` and closely related admission errors map as follows; diagnostic strings
never supply authority:

| Condition | Result |
| --- | --- |
| Missing/revoked provision, changed UID or capsule, stale package generation, or provision/manifest binding mismatch | `capability-denied` |
| Unknown requested worker ID, or ID that is not a declared `compute-worker` in an otherwise valid provision | `no-such-worker` |
| Worker bytes/hash, import/export signature, memory shape or ABI version malformed or mismatched during package validation | `worker-invalid` |
| Invalid positive-count, memory page or descriptor input | `invalid-input` |
| Finite policy or current reservation denies otherwise valid admission | `quota` |
| Owner/publication cancellation races an admitted open | `cancelled`; retain exact startup-worker custody until settlement |
| Compile/instantiate/trap or forced-interruption failure after valid admission | `worker-failed`, unless the more specific validation or cancellation case applies |

A bad package digest caught before publication is a publication failure and
must not become an authorized `open`. If a live provision disagrees with its
activation manifest, the authority mismatch is `capability-denied`; a worker
that is genuinely malformed inside otherwise authenticated matching package
bytes is `worker-invalid`.

## Raw worker ABI 1

A worker is a core-Wasm module, not a Component. It imports exactly one
`astrid_compute.memory`: shared memory32 or memory64 with a declared maximum.
Both address widths are bounded by the same finite admitted byte allowance;
memory64 is not permission to allocate an unbounded address space. It exports
`astrid_compute_abi_version: () -> i32` returning `1` and
`astrid_compute_run: (i32, i64, i64, i64) -> i32`. Other exports may be
present, but only the known import allowlist below is admitted. The chosen
memory width must match the worker's declared memory and any mutable stack
pointer export. Malformed,
duplicate, incorrectly typed and foreign-module imports fail before execution.

The first 64 shared-memory bytes are a conventional control header. Its fields
are unsigned little-endian: numeric magic `0x41534331` as `u32` at offset 0
(memory bytes `31 43 53 41`, not an ASCII string comparison);
version 1 as `u32` at 4; atomic cancellation generation as `u32` at 8;
offsets 12–15 reserved; admitted worker count as `u64` at 16; current memory
bytes as `u64` at 24; offsets 32–63 reserved. The host initializes it and
updates cancellation and growth fields with aligned atomic operations. A host
on a non-little-endian architecture must preserve this wire layout or reject
the worker; native atomic layout is not a portable serialization shortcut.
The worker can still write shared memory, including this header. These fields
are advisory to guest code and never authorize, meter or prove cancellation.
Controller descriptors start at 64 or later; `asset_read` cannot target the
header. Cancellation increments the generation and wakes transformed
`memory.atomic.wait32`/`wait64` waiters, but forced engine interruption remains
the authority when a worker ignores it or parks elsewhere. `memory.atomic.notify`
must preserve ordinary synchronization behavior while admitting cancellation.

Workers using a shared LLVM-style mutable stack may export a trio together:
`__stack_pointer` as mutable `i32` for memory32 or `i64` for memory64,
`astrid_compute_stack_reserve_bytes: () -> i32`, and
`astrid_compute_stack_stride_bytes: () -> i32`. The host validates a positive
reserved arena, positive stride, divisibility, bounds and an index-specific
stack slot before each instance runs. All three absent is permitted; a partial
or malformed trio is rejected. An implementation must not let concurrent
workers alias a declared private stack.

The first public ABI-1 **host must provide** these four functions. A worker
may import any subset, including none, with exactly these signatures:

| Import | Signature | Meaning |
| --- | --- | --- |
| `fuel_remaining` | `() -> i64` | Remaining fuel for the active job; `-1` means no active job or a failed observation, never unlimited fuel. |
| `asset_count` | `() -> i32` | Number of ordered admitted immutable assets; may be read before a job. |
| `asset_size` | `(i32) -> i64` | Byte length by index; negative on invalid index; may be read before a job. |
| `asset_read` | `(i32, i64, i64, i64) -> i32` | Copy from asset index/source offset into shared-memory destination/length. |

`asset_read` is callable only during a job, cannot overwrite the reserved
header, caps one call at 64 KiB, checks signed conversions and overflow, and
charges `64 + requested-byte-count` fuel before copying. It returns 0 on
success, or -1 invalid index, -2 invalid range, -3 excessive length, -4
insufficient fuel, -5 inactive phase. A destination-memory failure discovered
after charging may still return -2 with fuel consumed; callers must not infer
refund from an error code.
The host may reject any worker with an unsupported import even if its ABI
version function returns 1. Assets provide input bytes, not ambient host IO.

The original unmerged draft said workers import no functions. Main and
v2026.10.1 have no compute host; that experimental host has no public
compatibility promise. The first supported ABI-1 host must implement all four
functions while accepting workers that use none. An installer must reject a
package requiring compute on a runtime without this host contract *before*
publication, with a useful upgrade diagnostic. ABI 1 here is the first public
worker ABI, not a silent extension to a shipped one.

## Resource bounds, cancellation and settlement

Admission intersects finite process-wide worker, shared-memory, artifact,
archive-parser, table and job-fuel limits with principal policy and current
reservations. Every opened group charges its principal and the shared host
pool. Omitted descriptor fuel resolves to a finite share from effective
principal fuel-rate and job policy, narrowed to unspent capacity; an explicit
positive limit cannot widen policy. Zero remaining capacity or explicit zero
denies admission. A read import also consumes job fuel. Module instantiation,
its start function, ABI export calls and stack setup must have a separately
finite fuel budget as well as a startup timeout. The current experimental
runtime seeds startup with `u64::MAX` fuel: its 30-second interrupt timeout is
useful but does not satisfy this finite-fuel requirement. The implementation
may impose stricter bounds, but must never interpret omitted fuel or
`fuel_remaining() == -1` as unlimited execution.

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

Audit records must name the authenticated principal UID, capsule, exact package
generation, worker ID/BLAKE3 digest, ordered asset IDs/BLAKE3 digests, admitted
bounds, submit/cancel/terminal outcome and reason. Asset contents and descriptor
bytes stay out of audit records. WIT `group-info` and `accounting` do not carry
UID, generation, worker digest or asset inventory; their counters cannot by
themselves prove authorization. The host-stamped admission/audit record holds
those facts independently of guest-writable memory.

## Hosted and standalone implementations

Hosted Astrid relies on the host OS, sandbox configuration, storage and Wasm
engine for process isolation, scheduling and persistence. The same WIT and
worker ABI can be implemented by a standalone Astrid scheduler, but this RFC
does not make hosted deployment a verified-boot or formally verified kernel.
Conversely, seL4 kernel proofs would not prove this controller, Wasmtime
implementation, package registry, worker algorithm or Linux guest correct.
Neither implementation may silently substitute native host processes for
admitted worker modules. The authority and accounting semantics must remain
observable across both deployments.

## Conformance evidence before activation

Tests must include: exact package and generation binding; alias reuse with a
new principal UID; worker/asset reorder or hash mismatch; path substitution;
the wrong capsule and system residency; old no-asset and new asset-enabled
workers; malformed imports and ABI versions; both bounded memory32 and
memory64, including >4-GiB memory64 addressing; stack-slot separation and
header mutation; memory/descriptor overflow and grow ceiling; asset read
tariff/errors; cross-principal resource isolation; finite omitted job and
module-start fuel; cancellation during startup, atomic wait/notify and after
submission; post-cancel read versus closed grow; trap and forced interruption;
and retained reservation until exact native worker join. A packaged Linux Realm
must then install this contract, open its group, boot its real guest and
exercise shutdown/restart. Source tests and a synthetic compatibility fixture
do not substitute for that installed journey. Translated/Rosetta x86 processes
do not supply native Intel proof; that platform remains unverified until run
on native hardware.

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

- Implement the `unsupported-compute-host` installer diagnostic and verify its
  rejection precedes publication; the name here is normative, not evidence that
  current installers emit it.
- Replace experimental `u64::MAX` startup fuel with a measured finite bound
  before claiming conformance, without breaking legitimate signed data setup.
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
