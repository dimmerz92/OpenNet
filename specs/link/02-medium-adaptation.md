# 02. Medium Adaptation Shims

## 1. Purpose

This section defines the boundary between the OpenNet link service and arbitrary physical or logical media. It establishes three things: the **shim interface**, the contract a shim MUST satisfy to produce a compliant link; the **capability descriptor**, the structured report of medium properties a shim MUST produce and the link service consumes; and the **Enumerate-Links** primitive by which the Network layer discovers the node's set of links.

This section does not redefine any cross-cutting invariant from section 00 or any service contract from section 01. Where a later section defines a mechanism (framing, addressing, quality), that section is authoritative for the mechanism; this section is authoritative for the shim interface, the descriptor, the validation floor for descriptor values, and link enumeration.

The native API of a medium is out of scope (LINK-OVR-020). This specification defines what sits on the OpenNet side of the boundary, not what a medium must expose. New media are supported by writing new shims, not by revising this specification (LINK-OVR-021).

Requirement IDs in this section use the `LINK-MED-NNN` prefix.

## 2. Shim interface

### 2.1 Form of the interface

A medium is adapted into a compliant link by a shim that implements the shim interface defined in this section (LINK-MED-001), restating LINK-OVR-019 and LINK-SVC-024 for this section.

The shim interface is a logical contract. Each operation is defined normatively by its logical inputs, logical outputs, and the constraints and ordering rules that govern its invocation; it is not defined by any particular syntactic function shape (LINK-MED-002). An implementation conforms by exposing operations that carry the named inputs and outputs and obey every stated constraint and ordering rule, regardless of how those operations are expressed in the implementation's language or runtime. No representation is mandated or privileged; the rule of LINK-SVC-077 applies to the shim interface as it does to the link service primitives.

The link service MUST NOT call into a medium's native API except through the shim interface operations defined here (LINK-MED-003), restating LINK-SVC-025.

Operations are listed below by logical direction. **Downward** operations are invoked by the link service on the shim. **Upward** indications are emitted by the shim to the link service. A compliant shim MUST support every operation in this section (LINK-MED-004).

### 2.2 Downward: Shim-Start

The `Shim-Start` operation requests that the shim bring its medium binding into use and begin observing medium state (LINK-MED-005).

**Logical inputs:**

- `link_id`: the host-local handle the link service has assigned to this link, opaque to the shim and to the medium. The shim treats it as an opaque correlation token.

**Logical outputs** (exactly one per invocation):

- `started`: the shim has accepted the binding and will begin observing the medium. It has not yet asserted that the medium is up.
- `rejected-resource`: host-local resources for an additional link are exhausted.
- `rejected-invalid`: the shim cannot produce a valid capability descriptor for this medium.
- `rejected-invalid-state`: the `link_id` has already been started or was previously retired.

**Constraints and ordering:**

- The link service MUST NOT interpret a `started` result as asserting that the medium is up (LINK-MED-055). Medium availability is surfaced by the shim state-change indication (section 2.7) and mapped by the lifecycle governor (section 01.5.4).
- The link service MUST invoke `Shim-Start` at most once per `link_id` (LINK-MED-056); `Shim-Stop` (section 2.3) permanently retires the binding and identifier. If `Shim-Start` is invoked for an existing or retired `link_id`, the shim MUST return `rejected-invalid-state` without altering any binding (LINK-MED-057).

### 2.3 Downward: Shim-Stop

The `Shim-Stop` operation requests that the shim tear down its medium binding and release its host-local resources (LINK-MED-006).

**Logical inputs:**

- `link_id`: the handle assigned at `Shim-Start`.

**Logical outputs** (exactly one per invocation):

- `stopped`: the binding has been released and permanently retired.
- `unknown-binding`: no live binding exists for `link_id`, including after a completed stop.
- `rejected-invalid-state`: the binding is `Available` and has not completed the quiesce barrier.

**Constraints and ordering:**

- `Shim-Stop` on an `Available` binding MUST follow a completed down-purpose `Shim-Quiesce`. A stop from pre-up `Unavailable` or `Unavailable-Pending-Up` does not require another barrier if the shim already provides the equivalent cut-off guarantee (LINK-MED-079).
- After `Shim-Stop`, the shim MUST NOT emit further receive or state-change indications for that `link_id` (LINK-MED-058).
- `Shim-Stop` is the permanent retirement signal referenced by LINK-SVC-062. The `link_id` MUST NOT be started again or reused during the current host run; rediscovery of the same medium uses a new `link_id` (LINK-MED-059).

### 2.4 Downward: Shim-Send

The `Shim-Send` operation hands a generic frame, as defined in section 03, to the shim for transmission on the medium (LINK-MED-007).

**Logical inputs:**

- `link_id`: the handle of the link.
- `frame`: a generic frame per section 03, carrying one or more payload fragments and link-layer addressing per sections 04 and 05.

**Logical outputs** (exactly one per invocation):

- `accepted`: the frame was accepted for transmission. Acceptance indicates the shim has taken responsibility for attempting transmission, not that the frame has been or will be delivered.
- `rejected-queue-full`: the shim's bounded transmission resources are exhausted. This rejection is transient.
- `rejected-link-unavailable`: the shim cannot transmit (for example, the medium is down, or the shim cannot transmit for a reason reported through the descriptor; see section 4.4). Direction-specific send rejection is handled at the service boundary per section 4.4 and LINK-SVC-054, not by the shim.
- `rejected-malformed`: the generic frame fails the validations defined in section 03.
- `unknown-binding`: no live binding exists for `link_id`.

**Constraints and ordering:**

- `Shim-Send` is asynchronous with respect to medium transmission, restating LINK-SVC-033 at the shim boundary.
- The shim MUST reject `Shim-Send` as `rejected-link-unavailable` while the binding is not up or while a quiesce barrier is in progress. It MUST return `unknown-binding` for an unknown or retired identifier (LINK-MED-080).
- The shim MUST translate the generic frame to the medium's native framing on send, restating LINK-OVR-022 (LINK-MED-008). The translation MUST preserve the opaque payload byte sequence exactly; the shim MUST NOT inspect, parse, or act on payload contents (LINK-MED-009), restating LINK-OVR-002 for the shim.
- For a stream-oriented medium, the shim MUST perform stream-to-datagram adaptation as defined in section 3 before transmission (LINK-MED-010).

### 2.5 Downward: Shim-Quiesce

The `Shim-Quiesce` operation establishes a binding-scoped ownership cut-off before `Link-Down` or before a reduced service size becomes authoritative (LINK-MED-081).

**Logical inputs:**

- `link_id`: the handle of the binding.
- `purpose`: exactly one of `down` or `size-change`.

**Logical outputs** (exactly one per invocation):

- `quiesced`: no frame that remained uncommitted at the cut-off can begin native transmission. For `size-change`, this result is accompanied by `returned_frames`, containing every uncommitted generic frame removed from shim ownership, and every frame committed before the cut-off has completed native transmission. For `down`, uncommitted frames are cancelled, `returned_frames` is empty, and every reception admitted before the receive cut-off has been emitted as `Shim-Receive` or discarded under the observable-discard rules.
- `failed-timeout`: the barrier failed or exceeded its finite deadline. Shim admission remains closed and the binding must be forced down and retired under LINK-SVC-104.
- `unknown-binding`: no live binding exists for `link_id`.
- `rejected-invalid-state`: no barrier is applicable in the binding's current state.

**Constraints and ordering:**

- Once `Shim-Quiesce` begins, the shim MUST reject new `Shim-Send` operations for the binding. After a successful `size-change` barrier, sending MAY resume under the new coherent tuple after `Link-Changed`. After a successful `down` barrier, the binding remains quiescing until `Shim-Stop`; it cannot resume and no new `Shim-Send` is permitted (LINK-MED-082).
- For `down`, a frame irreversibly committed to native transmission before the cut-off MAY complete after the barrier. For `size-change`, every pre-cut-off committed frame MUST complete before `quiesced` is returned. The applicable adapter profile MUST define the commit boundary precisely enough to distinguish committed from cancellable work (LINK-MED-083).
- A `quiesced` result MUST NOT be returned until every uncommitted frame has been cancelled for `down` or returned exactly once in `returned_frames` for `size-change`. `returned_frames` preserves every occurrence and its exact bytes and MUST NOT coalesce byte-identical frames; its ordering is unspecified. The shim MUST NOT retain cancelled or returned work and MUST NOT carry it into a later availability epoch (LINK-MED-084).
- For a `down` barrier, the shim MUST establish a receive cut-off. It MUST emit every admitted pre-cut-off `Shim-Receive` before the barrier completes or discard it under the section 06 observability mechanism. It MUST drain, cancel, or irrevocably fence every incomplete stream prefix, native subdivision or aggregation context, delayed native callback, byte, native unit, and metadata item belonging to the ending episode. No old-side input may contribute to a later `Shim-Receive`; later receptions are discarded and never retained for a later episode (LINK-MED-098).
- Every barrier MUST have a finite implementation-defined or adapter-profile-defined deadline (LINK-MED-100). If it cannot complete within that deadline, the shim MUST return `failed-timeout`, keep send and receive admission closed, and discard or irrevocably fence every uncommitted send, receive context, native callback, and metadata item. A failed size change MUST NOT emit or authorise the reduced descriptor. The service then applies LINK-SVC-104 (LINK-MED-100).

### 2.6 Upward: Shim-Receive indication

The shim emits a `Shim-Receive` indication when one or more generic frames have been reconstructed from a medium reception (LINK-MED-011).

**Logical outputs** (emitted by the shim to the link service):

- `link_id`: the handle of the link on which the reception occurred.
- `frame`: a generic frame per section 03, reconstructed from the medium's native reception.
- `recv_meta`: shim-reported receive quality hints; MAY be empty. The normative contents of `recv_meta` at the service boundary are defined in section 06.

**Constraints and ordering:**

- The shim MUST reconstruct a generic frame from the medium's native framing on receive, restating LINK-OVR-022 (LINK-MED-012). The reconstruction MUST preserve the opaque payload byte sequence as received; the shim MUST NOT inspect, parse, or act on payload contents (LINK-MED-009).
- A single medium reception MAY yield zero, one, or several `Shim-Receive` indications, depending on how the native reception maps to generic frames (LINK-MED-060). For a stream-oriented medium, section 3 governs reconstruction.
- A `Shim-Receive` indication carrying a frame that fails the validations of section 03 MUST be discarded by the link service; the discard MUST NOT affect other receptions or links (LINK-MED-061), restating LINK-SVC-069 and LINK-SVC-070.
- The link service MUST admit a valid `Shim-Receive` to `Receive` only during the current `Available` episode. It MUST apply the pre-up, down-cut-off, observable-discard, and epoch rules of LINK-SVC-091 and LINK-SVC-092 (LINK-MED-094).

### 2.7 Upward: Shim-State-Changed indication

The shim emits a `Shim-State-Changed` indication when the medium's availability or a reported capability changes (LINK-MED-013).

**Logical outputs** (emitted by the shim to the link service):

- `link_id`: the handle of the link.
- `kind`: one of `up`, `down`, `changed`, or `barrier-failed`.
- `descriptor`: when `kind` is `up` or `changed`, the shim's current capability descriptor as defined in section 4. When `kind` is `down`, `descriptor` MAY be omitted.
- `barrier_purpose`: present only for `barrier-failed`, with value `down` or `size-change` identifying the failed internal barrier; `descriptor` MUST be omitted.

**Constraints and ordering:**

- The lifecycle governor (LINK-SVC-055) maps `up`, `down`, and `changed` indications into `Link-Up`, `Link-Down`, and `Link-Changed` under section 01.5, and maps `barrier-failed` into the forced retirement path of LINK-SVC-104. An `up` does not cause `Link-Up` if its descriptor fails validation (LINK-MED-062).
- The shim MUST NOT silently retry a medium connection or conceal state changes from the link service (LINK-MED-063), restating LINK-SVC-050. Whether the shim internally attempts re-establishment is a shim behaviour; any state the shim wishes to surface MUST be surfaced through this indication.
- A shim MAY emit `kind` of `down` after it has emitted `up` for the binding and MUST NOT depend on knowing whether the lifecycle governor mapped that `up` to `Link-Up` (LINK-MED-064). Before emitting `down`, the shim MUST establish the same ownership cut-off guaranteed by a successful down-purpose `Shim-Quiesce`. When the governor admits the indication, it MUST apply that same cut-off to service admission and service-owned work before emitting `Link-Down` (LINK-MED-085).
- Before emitting `changed` with a descriptor that reduces `maximum_frame_size` or otherwise causes a reduced service size, the shim MUST close shim admission and drain all prior shim-owned work under the old tuple, so no returned frames remain, before emitting the indication. When the governor admits the indication, it MUST establish the service-side size cut-off, linearise concurrent `Send` calls, and classify service-owned work before emitting `Link-Changed` (LINK-MED-095).
- The service MUST accept `changed` as a capability transition only after availability for that binding has been established. A `changed` indication received before first availability or while the service state is not `Available` MUST be discarded in isolation and MAY be diagnosed; a shim correcting a pre-availability descriptor MUST emit another `up` (LINK-MED-102).
- If a shim-originated down or size-change barrier fails or exceeds its finite deadline, the shim MUST close shim admission, fence uncommitted send and receive work under LINK-MED-100, and emit `barrier-failed` for the exact binding and purpose (LINK-MED-105). The governor MUST NOT coalesce or discard this indication. It MUST close service admission and apply LINK-SVC-104. A failed size change MUST NOT emit `changed` or publish the reduced tuple.
- All upward indications emitted for the same binding MUST be delivered to the link service in emission order. In particular, every pre-cut-off `Shim-Receive` that is not discarded MUST be delivered before the corresponding `down`; an indication from another binding MAY be delivered in any relative order (LINK-MED-099).
- The lifecycle governor MAY coalesce an `up, down` sequence without service lifecycle events only when the entire sequence occurs before any authoritative `Available` episode has been exposed, and MAY coalesce redundant same-state reports that cross no availability boundary (LINK-MED-065). Once availability has been exposed, an accepted `down` MUST produce `Link-Down`, and a later accepted `up` for the continuing temporary-loss binding MUST produce `Link-Up` with a new epoch. Every permitted coalescing remains bounded, reflects the latest state within a finite implementation-defined interval, and is observable through the mechanism required by sections 06 and 08.

Note: a shim whose `up` indication failed validation remains able to emit a further `up` carrying a corrected descriptor. If it emits `down` first, the lifecycle governor may coalesce the unexposed `up, down` sequence without emitting either service event.

### 2.8 Downward: Shim-Describe

The `Shim-Describe` operation returns the shim's current capability descriptor (LINK-MED-014).

**Logical inputs:**

- `link_id`: the handle of the link.

**Logical outputs** (exactly one per invocation):

- `descriptor`: the shim's current capability descriptor per section 4.
- `unknown-binding`: no live binding exists for `link_id`.

**Constraints and ordering:**

- `Shim-Describe` MUST NOT change medium state and MUST NOT have side effects on any link or flow (LINK-MED-015).
- The descriptor returned by `Shim-Describe` is shim-owned input that the link service validates before use per section 5.
- Each successful `Shim-Describe` has one binding-local logical observation point ordered with descriptor-bearing `up` and `changed` indications for that binding (LINK-MED-101). Indications logically before the observation are delivered before the result can become authoritative, and indications logically after it are delivered afterwards. A describe result MUST NOT replace descriptor state installed by a logically later indication.

Note: `Shim-Describe` is expected to be inexpensive and to return without blocking on medium I/O; the link service may invoke it on latency-sensitive paths.

### 2.9 Binding operation states

For operation validation, a shim binding is logically in one of `bound-down`, `bound-up`, `quiescing`, or `retired` (LINK-MED-096). `started` creates `bound-down`; an emitted `up` enters `bound-up`; a shim-originated temporary-loss `down` enters `bound-down`; `Shim-Quiesce` enters `quiescing`; a successful size change returns to `bound-up` after `Link-Changed`, while a down-purpose barrier remains `quiescing` until `Shim-Stop`; and `Shim-Stop` enters terminal `retired`.

`Shim-Send` is valid only in `bound-up` outside a quiesce barrier. `Shim-Quiesce` is valid for a live binding when its purpose can establish a new cut-off. `Shim-Stop` is valid in `bound-down` or after a down-purpose barrier in `quiescing`. `Shim-Describe` is valid for any live binding. Calls outside these states MUST return the safe rejection defined by that operation, without changing binding, medium, or other-link state (LINK-MED-097). The implementation MAY retain a bounded identifier tombstone or use allocator state to distinguish a retired identifier; in either case, it MUST prevent a retired identifier from creating another binding during the host run.

## 3. Stream-to-datagram adaptation

A medium that is natively stream-oriented is adapted to a datagram link by its shim (LINK-SVC-003). The datagram unit the link service sees is the generic frame defined in section 03.

A shim for a stream-oriented medium MUST translate the byte stream into complete generic frames on receive and generic frames into the byte stream on send (LINK-MED-016). The unit exposed at the shim boundary is always one complete generic frame; the medium-side translation MAY use a framing envelope or native record mechanism and MUST NOT expose any partial frame upward (LINK-MED-017).

A stream medium with no established native record and recovery convention MUST use the canonical bare-stream profile defined with the frame format in section 03 (LINK-MED-066). That profile MUST be self-delimiting and resynchronising. A stream medium with an established native adapter profile MAY use that profile instead, provided both peers use the same profile and it satisfies the safety requirements of this section (LINK-MED-086).

Every stream adapter profile MUST define a finite parser-buffer bound no greater than the global maximum frame size plus the profile's fixed envelope overhead, a finite incomplete-frame dwell or progress bound, bounded parsing work under repeated malformed prefixes, deterministic discard and resynchronisation, and isolation from other links (LINK-MED-087). A malformed prefix MUST NOT by itself make the link unavailable. If an allowed profile cannot safely resynchronise, it MUST define the precise reset condition and resulting lifecycle indication (LINK-MED-088).

For a natively datagram- or packet-oriented medium, a shim MAY segment one generic frame across multiple native units, aggregate generic frames into one native unit, or reassemble native units below the shim boundary (LINK-MED-018). The applicable adapter profile MUST make the translation interoperable between peers. Shim-private adaptation MUST have finite byte, context, work, and lifetime bounds, MUST abandon incomplete state within its lifetime bound, MUST reconstruct each completed generic frame exactly, and MUST NOT expose a partial frame (LINK-MED-089). A native input event that cannot contribute to a valid generic frame within those bounds MUST be discarded without affecting other adaptation contexts or links (LINK-MED-067). On send, a generic frame presented to `Shim-Send` that exceeds the link's reported `maximum_frame_size` MUST be rejected as `rejected-malformed` (LINK-MED-068). Fragmentation of Network-layer payloads is performed inside the link service per LINK-SVC-006 and is not a shim responsibility; native-unit subdivision below the shim boundary is distinct from that fragmentation.

A shim MUST NOT require the link service to know whether the underlying medium is stream- or datagram-oriented (LINK-MED-019): the link service issues `Shim-Send` and observes `Shim-Receive` in terms of generic frames for both classes of medium.

## 4. Capability descriptor

### 4.1 Representation

The capability descriptor is a logical structure (LINK-MED-020). Section 02 normatively defines the descriptor's field semantics, types, allowed values, and validation and consistency rules. The in-host representation of the descriptor is implementation-defined: an implementation MAY represent it as a struct, record, map, message, or any other structure that carries the named fields with the named semantics (LINK-MED-020). The byte-order and canonical-serialisation invariants of section 00.8 (LINK-OVR-009, LINK-OVR-011) do NOT apply to the capability descriptor (LINK-MED-021), because the descriptor is host-local input (LINK-OVR-027) that never crosses a medium and is consumed through primitives that are representation-agnostic (LINK-SVC-077).

### 4.2 Core field-set

The descriptor carries a fixed normative core field-set for v1 (LINK-MED-022). The core fields are:

Each named core field has at most one logical occurrence. A descriptor representation containing more than one occurrence of the same core field is structurally malformed and follows the hard-failure path of LINK-MED-041 (LINK-MED-103). This rule does not prohibit duplicate `extension_id` values in the `extensions` collection.

- `maximum_frame_size` (integer; required): the maximum generic frame size, in bytes, the shim can present in the send direction. For a send-capable direction, this is the value reported as the link's MTU upward (LINK-SVC-007) and from which the link's maximum packet size is derived per section 03; for a `receive-only` link the value is interpreted per section 4.4 and `Send` is rejected per LINK-SVC-054.
- `receive_max_frame_size` (integer; optional, defaults to `maximum_frame_size` when absent): the maximum generic frame size the shim can present in the receive direction. Informational; the Network layer MAY consult it. Ignored for `send-only` links.
- `one_way_direction` (enumeration, required): one of `bidirectional`, `send-only`, `receive-only`, or `asymmetric`, governing the send and receive availability rules of section 01.5.3 (LINK-SVC-052, LINK-SVC-083).
- `reliable_delivery` (boolean; optional, absent denotes false): in the send direction, the medium provides the reliable-delivery semantic defined in section 06 for accepted `Send` operations.
- `ordered_delivery` (boolean; optional, absent denotes false): in the send direction, the medium preserves the ordering semantic defined in section 07 for accepted `Send` operations.
- `duplicate_suppression` (boolean; optional, absent denotes false): in the receive direction, the shim or medium provides the duplicate-suppression semantic defined in section 07 for frames eligible to become `Receive`.
- `native_retransmission` (boolean; optional, absent denotes false): in the send direction, the shim or medium performs the retransmission semantic defined in section 06.
- `broadcast_support` (boolean; optional, absent denotes false): in the send direction, the medium supports a link-layer broadcast destination.
- `multicast_support` (boolean; optional, absent denotes false): in the send direction, the medium supports a link-layer multicast destination. The normative link-layer multicast address semantic is defined in section 05; this section defines only the capability flag.
- `full_duplex` (boolean; optional, absent denotes false): as a whole-link property, the medium supports simultaneous send and receive. Informational.
- `intermittence` (boolean; optional, absent denotes false): as a whole-link property, the medium is intermittent. Affects lifecycle expectations per LINK-SVC-050.
- `latency_class` (informational token; optional; the sentinel `unknown` denotes unreported): an implementation-defined whole-link hint describing latency conservatively across supported directions.
- `throughput_class` (informational token; optional; sentinel `unknown` denotes unreported): an implementation-defined whole-link hint describing throughput conservatively across supported directions.
- `energy_class` (informational token; optional; sentinel `unknown` denotes unreported): an implementation-defined whole-link hint describing energy use conservatively across supported directions.

The three class fields carry no normative vocabulary: they are useful for link selection only when the shim and the Network layer share an out-of-band vocabulary, and v1 deliberately defines none. A core capability needed for correctness in both directions MUST use explicit direction-qualified core fields; it MUST NOT depend on an extension to express the missing direction (LINK-MED-090).

An implementation MUST NOT infer a capability the descriptor does not report (LINK-MED-069), restating LINK-SVC-020 at the descriptor definition.

### 4.3 Extension fields

A shim MAY report additional properties beyond the core field-set (LINK-MED-023). Extension fields are carried in an optional `extensions` collection of zero or more `(extension_id, value)` pairs. `extension_id` is an opaque value that identifies the extension; this specification does not enumerate extension identifiers in v1, and a value's structure is owned by its defining extension, not by this specification. A descriptor consists of the core field-set of section 4.2 and the `extensions` collection and nothing else; a descriptor carrying any other component is invalid under section 5 (LINK-MED-070).

The link service MUST distinguish core fields from extension fields (LINK-MED-024). The Network layer MAY observe extension fields but MUST NOT depend on them for correctness (LINK-MED-025). The link service MUST NOT fail validation of a descriptor solely because an `extension_id` is unrecognised (LINK-MED-026); unknown extensions are forward-compatible and preserved. An implementation MUST bound the number or aggregate size of extension fields it preserves; the bound is implementation-defined (LINK-MED-027). If the bound is exceeded, the implementation MAY drop excess extensions but MUST NOT fail the descriptor solely on account of extension volume. The bound MUST be small enough that the per-link state owned by the link service, including the authoritative descriptor copy, remains within the resource bound required by LINK-MED-054 under adversarial input. The authoritative descriptor copy maintained by the link service per section 01.6.3 includes the preserved extension fields. Duplicate `extension_id` entries within one descriptor are not forbidden (LINK-MED-021 disapplies the duplicate prohibition of LINK-OVR-011); their handling is implementation-defined. For `Link-Changed` `change_set` purposes (LINK-SVC-043), the `extensions` collection is treated as the single field `descriptor.extensions` (LINK-MED-071).

### 4.4 Direction-specific fields

For a link whose `one_way_direction` is `send-only`, the link service MUST NOT emit `Receive` (restating LINK-SVC-053); `receive_max_frame_size`, if present, MUST be ignored (LINK-MED-028). For a link whose `one_way_direction` is `receive-only`, the link service MUST reject `Send` as `rejected-direction-unsupported` (restating LINK-SVC-054). For such a link, `maximum_frame_size` describes the receive-capacity bound the shim uses when reconstructing generic frames, and the reported `maximum_frame_size` MUST satisfy the same bounds as for bidirectional links (LINK-MED-029). For an `asymmetric` link, the descriptor reports `maximum_frame_size` for the send direction and `receive_max_frame_size` for the receive direction when they differ; every other core field has the directional scope stated in section 4.2, and the rules of LINK-SVC-029 (`Send`) and LINK-SVC-034 (`Receive`) apply in their respective directions (restating LINK-SVC-083).

## 5. Validation and fail-closed

### 5.1 Validation floor

Before any descriptor value is used to size state or select link behaviour, an implementation MUST process the descriptor in this order: validate its structure and required fields; normalise optional fields and optional-field consistency failures to their safe defaults; validate the resulting descriptor; then derive service values and replace the authoritative copy (LINK-MED-030), restating LINK-OVR-027 and LINK-SVC-026. The same order applies before first availability and after availability (LINK-MED-091). The capability descriptor is host-local input, trusted within the host trust boundary (LINK-OVR-027): the shim is trusted host-local code. Its values and indications are nonetheless validated because they may derive from an adversarial medium or faulty code. The semantic truth of a well-formed report cannot be established by validation and remains accepted residual risk; section 01.2.3 carries the mitigating note.

Descriptor input, traversal, nesting, normalisation, comparison, and `change_set` construction MUST be subject to finite implementation-defined size, depth, and work budgets that prevent one binding from starving another (LINK-MED-104). If the core or descriptor structure cannot be validated within those budgets, the descriptor is a hard structural failure. Extension processing stops at its budget and drops unprocessed or excess extensions under LINK-MED-027 without failing an otherwise valid descriptor. Budget exhaustion and dropping MUST have a deterministic implementation-defined outcome and MAY be diagnosed.

Where a safety-relevant capability is absent or invalid, the implementation MUST fail closed and behave as if the capability were absent (LINK-MED-031), restating LINK-OVR-028 and LINK-SVC-027.

### 5.2 Field validation

`maximum_frame_size` MUST be an integer greater than or equal to the guaranteed minimum MTU and less than or equal to the global maximum frame size, both defined in section 03 (LINK-MED-032). A `maximum_frame_size` outside this range is invalid.

`receive_max_frame_size`, when present, MUST satisfy the same bounds as `maximum_frame_size` (LINK-MED-033), except on `send-only` links, where the field is ignored per LINK-MED-028. When absent, it defaults to `maximum_frame_size`.

`one_way_direction` MUST be present and be one of the four values listed in section 4.2; it has no default (LINK-MED-034). An absent or unrecognised `one_way_direction` is invalid.

For each boolean core field, an absent value denotes `false` (LINK-MED-035). A present value that is not a boolean MUST be normalised to `false` before descriptor validity is decided, so that no unsupported capability is advertised.

For each informational class field (`latency_class`, `throughput_class`, `energy_class`), an absent value or the sentinel `unknown` denotes unreported (LINK-MED-036). The link service MUST NOT assign a normative categorical meaning to a present non-`unknown` token; values are opaque tokens compared for equality only. The Network layer MAY consult them as hints but MUST NOT depend on them for correctness. A malformed present value MUST be normalised to `unknown` before descriptor validity is decided (LINK-MED-036).

### 5.3 Consistency rules

The following cross-field rules are applied during optional-field normalisation:

- If `one_way_direction` is `send-only`, `receive_max_frame_size` is ignored (section 4.4) and the descriptor is otherwise valid (restating LINK-MED-028).
- If `one_way_direction` is `receive-only`, `maximum_frame_size` describes the receive-capacity bound and MUST satisfy the bounds of section 03 (LINK-MED-033). For `receive-only` links, `maximum_frame_size` is the operative receive-capacity bound (LINK-MED-029); `receive_max_frame_size`, if present, remains subject to LINK-MED-033 and is otherwise informational.
- `broadcast_support` on a `receive-only` link MUST be normalised to `false`, because it is a send-direction capability (LINK-MED-037).
- `multicast_support` on a `receive-only` link MUST be normalised to `false` for the same reason (LINK-MED-038).
- `full_duplex` together with `one_way_direction` of `send-only` or `receive-only` MUST be normalised to `false` (LINK-MED-039).
- No further cross-field rule is defined in v1. An implementation MAY apply additional host-local diagnostics but MUST NOT reject or alter a descriptor solely on the basis of a rule not stated here (LINK-MED-040).

### 5.4 Invalid descriptor handling

A descriptor with a structural error, an unknown top-level component, or an absent or invalid required field is invalid and cannot be normalised into validity. The link service MUST NOT bring the link into `Available` for such a descriptor: the lifecycle governor MUST NOT emit `Link-Up`, and the link remains `Unavailable` (LINK-MED-041). Because the link has not emitted `Link-Up`, the lifecycle governor MUST NOT emit `Link-Down` for it (restating LINK-SVC-049). The link service MAY report the invalidity through an implementation-defined diagnostic channel (LINK-MED-042).

A descriptor reported by `Shim-State-Changed` with `kind` of `changed`, or returned by `Shim-Describe`, is processed through the same ordered normalisation and validation pipeline after a link is `Available` (LINK-MED-043). A hard failure affecting a required field or structure MUST be followed by the down barrier and `Link-Down` with `descriptor-invalid`.

An invalid required core field (`maximum_frame_size`, `one_way_direction`) in a post-`Available` descriptor cannot fail closed to a default and MUST be treated as affecting availability (LINK-MED-072). An invalid optional boolean is normalised to `false`, an invalid class token to `unknown`, an invalid `receive_max_frame_size` to its absent-derived default, and an optional-field consistency failure to the safe default of the offending optional capability (LINK-MED-073). The authoritative copy stores only the normalised descriptor. `Link-Changed` is emitted only when that normalised descriptor or its derived service values differ from the current coherent tuple (LINK-MED-073).

### 5.5 Indication provenance

Upward indications are shim-emitted input to the link service and are validated for provenance and structure before they are acted on. The link service MUST discard each of the following upward indications: an indication whose `link_id` is not bound to the emitting shim by an active `Shim-Start`; an indication for a `link_id` after `Shim-Stop`; a `Shim-State-Changed` indication whose `kind` is not one of the four values in section 2.7; and any structurally malformed indication (LINK-MED-074). A discard under this rule MUST NOT affect other links or receptions and MAY be reported through the diagnostic channel of LINK-MED-042.

## 6. Enumerate-Links primitive

The link service MUST provide an `Enumerate-Links` downward primitive by which the Network layer discovers the node's current set of links (LINK-MED-044). Link selection is not performed by the link service (restating LINK-SVC-065); this primitive exposes the inputs the Network layer uses to select, including the link set itself, which `Link-Query` (section 01.4.7) cannot return because `Link-Query` takes a `link_id` it presupposes.

**Logical inputs:** none; `Enumerate-Links` is invoked by the Network layer on the link service.

**Logical outputs:**

- `links`: a collection of zero or more per-link snapshots. Each snapshot carries, for one known `link_id`:
  - `link_id`;
  - `state`: the link's current lifecycle state per section 01.5;
  - `descriptor`: the current validated capability descriptor, which is `null`/absent when the link is `Unavailable` and has never validated a descriptor;
  - `epoch`: the generation value most recently reported by `Link-Up`, or an `unset` value when no `Link-Up` has occurred;
  - `mtu`: the currently reported MTU, or `unset` when the link has not reported one;
  - `max_packet_size`: the currently reported maximum packet size, or `unset` when the link has not reported one.

  For an `Unavailable` or `Unavailable-Pending-Up` link that has previously validated a descriptor, the snapshot carries the most recently validated descriptor; per LINK-MED-047 this value is stale and conveys no current guarantee. A `Retired` record MAY be included until its record is released. The returned collection may be realised as a snapshot copy or as a borrowed view.

**Constraints and ordering:**

- `Enumerate-Links` MUST NOT change link state and MUST NOT have side effects on any link or flow (LINK-MED-045), restating the side-effect rule of `Link-Query` (LINK-SVC-084) for the enumeration case.
- `Enumerate-Links` MUST return a snapshot for every currently known `link_id`, including links in `Unavailable`, `Available`, `Unavailable-Pending-Up`, and retained `Retired` states (LINK-MED-046). A retired record MAY be released and omitted, but its identifier remains unavailable for reuse during the host run.
- The snapshot for a link is current only while the link's `state` is `Available`, restating LINK-SVC-086 (LINK-MED-047). The Network layer SHOULD call `Link-Query` for the specific `link_id` when it requires a fresh value after the snapshot (LINK-MED-075).
- Each per-link snapshot MUST be internally coherent at one logical observation point. For the same consumer, lifecycle events logically preceding that point MUST be delivered before the result becomes observable, and events logically following it MUST be delivered afterwards. An `Available` snapshot establishes the same authority to invoke `Send` as `Link-Up` (LINK-MED-092). The collection need not be globally atomic: different per-link snapshots MAY have different logical observation points.
- A borrowed view MUST remain immutable for a defined borrow lifetime during which the caller can observe it (LINK-MED-093).
- Per-link snapshot ordering within the returned collection is implementation-defined and carries no semantics (LINK-MED-048).
- The number of links a node may host simultaneously is implementation-defined; an implementation MAY impose a finite upper bound (restating LINK-SVC-071). `Enumerate-Links` returns whatever links exist within that bound.

v1 defines exactly one link-enumeration primitive, `Enumerate-Links` (LINK-MED-049). The link service exposes no metrics-aggregator primitive in v1; per-link quality metrics are exposed through `Link-Query` and through `recv_meta` on `Receive`, whose normative form is defined in section 06. This does not prevent later Link Specification sections from adding mechanism-specific service primitives under LINK-SVC-096, including neighbour-observation primitives in section 05, nor an extension from defining a non-v1 enumeration mechanism.

## 7. Conformance for shims

A shim conforms to this specification if and only if it satisfies every applicable implementation requirement identified by a `LINK-MED-NNN` requirement ID within this section (LINK-MED-050). Conformance for a shim means, at minimum, that it:

- implements every operation in section 2 (LINK-MED-004),
- reports a descriptor that satisfies the validation and consistency rules of section 5 when its medium binding is one the shim intends to surface (LINK-MED-051),
- translates between the generic frame of section 03 and the medium's native framing on send and on receive without inspecting, parsing, or acting on payload contents (LINK-MED-008, LINK-MED-009, LINK-MED-012),
- surfaces medium state changes through the `Shim-State-Changed` indication (LINK-MED-013), and
- for a stream-oriented medium, performs the adaptation of section 3 (LINK-MED-016).

New media are supported by writing new shims; this specification is not revised to enumerate or admit media (LINK-MED-052), restating LINK-OVR-021. An implementation MAY ship shims for any finite subset of media; additional shims are implementation work and do not affect v1 compliance (LINK-MED-053), restating LINK-OVR-007.

Conformance to media-specific behaviour that is reported as absent in the validated descriptor is not required (LINK-MED-076), restating LINK-OVR-006. The capability descriptor is the sole determinant of which conditional requirements apply (LINK-MED-077), restating LINK-SVC-020 and LINK-SVC-021.

## 8. Limits, malformed input, and state bounds

A descriptor that exceeds an implementation's host-local bound on preserved extension fields is handled per LINK-MED-027; the descriptor is not failed. A descriptor that fails field validation or consistency is handled per section 5.4.

The link service MAY bound the number of shims it hosts simultaneously; the bound is implementation-defined (restating LINK-SVC-071). All per-link state owned by the link service or shim, including the authoritative descriptor copy, stream-parser state, native-unit subdivision state, and queued transmission state, is resource-bounded and MUST remain correct under adversarial input (LINK-MED-054), restating LINK-OVR-024 for this section. Adapter profiles supply the concrete bounds required by LINK-MED-087 and LINK-MED-089.

A `Shim-Send` returning `rejected-malformed` for a frame that fails section 03 validation MUST NOT affect any other frame, flow, or link (restating LINK-OVR-014). A `Shim-Receive` indication carrying an invalid frame is discarded per LINK-MED-061 and MUST NOT cause the link to become unavailable (restating LINK-OVR-014).

Sections 06 and 08 MUST define bounded lifecycle-event processing and shed behaviour under resource pressure, including the observability form of the coalescing permitted by LINK-MED-065 (LINK-MED-078).

The capability descriptors and bounds that govern `maximum_frame_size` and `receive_max_frame_size` are normative; this section references the guaranteed minimum MTU and the global maximum frame size defined in section 03 and does not restate their numeric values, which are owned by section 03.

## 9. Relationship to section 01

This section defines the downward adapter contract that section 01 references. Specifically:

- LINK-SVC-024 (a link is produced by a shim) is realised by LINK-MED-001.
- LINK-SVC-025 (the service does not call the native API except through the shim) is realised by LINK-MED-003.
- LINK-SVC-017 (a shim MAY report reliability, ordering, dedup, native retransmission) is realised by the corresponding boolean core fields of section 4.2.
- LINK-SVC-032 (broadcast governed by descriptor) is realised by the `broadcast_support` field.
- LINK-SVC-052 and LINK-SVC-083 (one-way and asymmetric links) are realised by `one_way_direction`.
- LINK-SVC-026 and LINK-SVC-027 (validation floor and fail-closed) are realised by section 5.
- LINK-SVC-036, LINK-SVC-043, and LINK-SVC-084 (descriptors carried in lifecycle primitives and `Link-Query`) consume the validated descriptor produced here.
- LINK-SVC-065 and LINK-SVC-066 (the link service does not select, but exposes the inputs for selection) are augmented here by the `Enumerate-Links` primitive (section 6).
- LINK-SVC-094 and LINK-SVC-095 (down and size-change ownership barriers) are realised by `Shim-Quiesce` and by the equivalent cut-off guarantee on a shim-originated `down`.

Where a statement in this section appears to conflict with a cross-cutting invariant from section 00 or a service contract from section 01, the section 00 or section 01 statement prevails; this section is in error.

## 10. Terminology used in this section

In addition to the terms defined in section 00, this section uses:

**shim interface**
The logical contract a shim MUST satisfy, defined in section 2 by operations with named logical inputs, outputs, and ordering constraints.

**quiesce barrier**
The binding-scoped ownership cut-off established by `Shim-Quiesce` or an equivalent shim-originated `down`.

**commit boundary**
The adapter-profile-defined point beyond which native transmission cannot be cancelled.

**capability descriptor**
The logical structure defined in section 4 reporting a medium's properties, produced by the shim and normalised and validated by the link service.

**core field**
A descriptor field defined normatively by v1 in section 4.2.

**extension field**
A descriptor field reported by a shim beyond the core set, identified by an opaque `extension_id`.

**adapter profile**
The shared medium-side framing, translation, bounds, and recovery contract used by peer shims.

**stream-oriented medium**
A medium whose native transport is a byte stream; adapted to a datagram link per section 3.

**datagram-oriented medium**
A medium whose native transport is discrete datagrams or packets.

**`Enumerate-Links`**
The downward primitive defined in section 6 by which the Network layer discovers the node's current link set.

**validation floor**
The ordered normalisation and validation process of section 5.

## 11. Requirement summary

The following normative requirements are defined in this section. Entries marked (restatement) restate a cross-cutting invariant from section 00 or a service contract from section 01 in the scope of this section; the section 00 or section 01 statement is authoritative, a conflict resolves in favour of that statement, and satisfying that statement is sufficient for conformance with the restated requirement. Entries marked (specification-suite invariant) bind the specification suite rather than a running implementation; see section 4 of section 00. This section is authoritative for the remaining requirements. Requirement IDs provide stable handles between this section and its conformance artefacts, including language-agnostic test vectors and automated conformance tests.

- **LINK-MED-001** (restatement of LINK-OVR-019 and LINK-SVC-024): A medium is adapted into a compliant link by a shim that implements the shim interface defined in section 2.
- **LINK-MED-002**: The shim interface is a logical contract defined by operations' logical inputs, outputs, and ordering/constraints, not by any particular syntactic function shape.
- **LINK-MED-003** (restatement of LINK-SVC-025): The link service MUST NOT call into a medium's native API except through the shim interface operations defined here.
- **LINK-MED-004**: A compliant shim MUST implement every operation in section 2.
- **LINK-MED-005**: `Shim-Start` requests that the shim bring its medium binding into use and begin observing medium state; its outputs are `started`, `rejected-resource`, `rejected-invalid`, and `rejected-invalid-state`.
- **LINK-MED-006**: `Shim-Stop` permanently retires a binding and releases its resources; its outputs are `stopped`, `unknown-binding`, and `rejected-invalid-state`.
- **LINK-MED-007**: `Shim-Send` hands a generic frame to the shim for transmission; its outputs are `accepted`, `rejected-queue-full`, `rejected-link-unavailable`, `rejected-malformed`, and `unknown-binding`.
- **LINK-MED-008** (restatement of LINK-OVR-022): The shim MUST translate the generic frame to the medium's native framing on send.
- **LINK-MED-009** (restatement of LINK-OVR-002): The shim MUST NOT inspect, parse, or act on payload contents; the opaque payload byte sequence MUST be preserved exactly on send and receive.
- **LINK-MED-010**: For a stream-oriented medium, the shim MUST perform stream-to-datagram adaptation per section 3 before transmission.
- **LINK-MED-011**: The `Shim-Receive` indication delivers a generic frame reconstructed from a medium reception, with optional `recv_meta`.
- **LINK-MED-012** (restatement of LINK-OVR-022): The shim MUST reconstruct a generic frame from the medium's native framing on receive.
- **LINK-MED-013**: `Shim-State-Changed` carries `link_id`, a kind of `up`, `down`, `changed`, or `barrier-failed`, and the descriptor or barrier purpose required by that kind; the governor maps it to lifecycle or forced-retirement behaviour.
- **LINK-MED-014**: `Shim-Describe` returns the shim's current capability descriptor or `unknown-binding`.
- **LINK-MED-015**: `Shim-Describe` MUST NOT change medium state and MUST NOT have side effects on any link or flow.
- **LINK-MED-016**: A stream shim MUST translate the byte stream into complete generic frames on receive and generic frames into the byte stream on send.
- **LINK-MED-017**: The unit exposed at the shim boundary is one complete generic frame; a medium-side envelope or record mechanism MAY be used but a partial frame MUST NOT be exposed upward.
- **LINK-MED-018**: A datagram or packet shim MAY segment, aggregate, and reassemble native units below the shim boundary under a shared adapter profile while exposing only complete generic frames.
- **LINK-MED-019**: A shim MUST NOT require the link service to know whether the underlying medium is stream- or datagram-oriented.
- **LINK-MED-020**: The capability descriptor is a logical structure; section 02 normatively defines field semantics, types, allowed values, and validation and consistency rules. The in-host representation is implementation-defined.
- **LINK-MED-021**: LINK-OVR-009 (big-endian) and LINK-OVR-011 (canonical serialisation) do NOT apply to the capability descriptor, because it is host-local and never crosses a medium.
- **LINK-MED-022**: The descriptor carries a fixed normative core field-set for v1 as listed in section 4.2.
- **LINK-MED-023**: A shim MAY report additional properties beyond the core field-set as extension fields.
- **LINK-MED-024**: The link service MUST distinguish core fields from extension fields.
- **LINK-MED-025** (specification-suite invariant): The Network layer MAY observe extension fields but MUST NOT depend on them for correctness.
- **LINK-MED-026**: The link service MUST NOT fail validation of a descriptor solely because an `extension_id` is unrecognised; unknown extensions are forward-compatible and preserved.
- **LINK-MED-027**: An implementation MUST bound the number or aggregate size of extension fields it preserves; the bound is implementation-defined; if exceeded, excess extensions MAY be dropped, but the descriptor MUST NOT be failed solely on account of extension volume; the bound MUST keep per-link state within the LINK-MED-054 bound under adversarial input.
- **LINK-MED-028**: For a `send-only` link, `receive_max_frame_size`, if present, MUST be ignored.
- **LINK-MED-029**: For a `receive-only` link, `maximum_frame_size` describes the receive-capacity bound and MUST satisfy the same bounds as for bidirectional links.
- **LINK-MED-030** (restatement of LINK-OVR-027 and LINK-SVC-026): Before using a descriptor, an implementation MUST validate structure and required fields, normalise optional fields and optional consistency failures, validate the result, derive service values, and then replace the authoritative copy.
- **LINK-MED-031** (restatement of LINK-OVR-028 and LINK-SVC-027): Where a safety-relevant capability is absent or invalid, the implementation MUST fail closed and behave as if the capability were absent.
- **LINK-MED-032**: `maximum_frame_size` MUST be an integer within the guaranteed minimum MTU and the global maximum frame size defined in section 03.
- **LINK-MED-033**: `receive_max_frame_size`, when present, MUST satisfy the same bounds as `maximum_frame_size`; when absent it defaults to `maximum_frame_size`.
- **LINK-MED-034**: `one_way_direction` MUST be present and one of `bidirectional`, `send-only`, `receive-only`, or `asymmetric`; it has no default.
- **LINK-MED-035**: For each boolean core field, an absent value denotes `false`; an invalid value MUST be treated as `false`.
- **LINK-MED-036**: For each informational class field, an absent value or the sentinel `unknown` denotes unreported; the link service MUST NOT assign a normative categorical meaning to a present non-`unknown` token (values are opaque, compared for equality only); a malformed present value MUST be treated as `unknown`; the Network layer MAY consult class tokens as hints but MUST NOT depend on them for correctness (the Network-layer clause binds the specification suite).
- **LINK-MED-037**: `broadcast_support` on a `receive-only` link MUST be normalised to `false`.
- **LINK-MED-038**: `multicast_support` on a `receive-only` link MUST be normalised to `false`.
- **LINK-MED-039**: `full_duplex` with `send-only` or `receive-only` MUST be normalised to `false`.
- **LINK-MED-040**: No further cross-field rule is normative in v1; additional host-local diagnostics MUST NOT reject or alter a descriptor solely under an unstated rule.
- **LINK-MED-041**: A structural error, unknown top-level component, or absent or invalid required field MUST prevent `Link-Up`; the link remains `Unavailable`.
- **LINK-MED-042**: An invalid-descriptor diagnostic MAY be reported through an implementation-defined channel; it is not a normative primitive.
- **LINK-MED-043** (restatement of LINK-SVC-045): A post-availability descriptor uses the same normalisation and validation pipeline; a hard structural or required-field failure MUST be followed by the down barrier and `Link-Down` with `descriptor-invalid`.
- **LINK-MED-044**: The link service MUST provide an `Enumerate-Links` downward primitive by which the Network layer discovers the node's current set of links.
- **LINK-MED-045** (restatement of LINK-SVC-084 for enumeration): `Enumerate-Links` MUST NOT change link state and MUST NOT have side effects on any link or flow.
- **LINK-MED-046**: `Enumerate-Links` MUST return a snapshot for every currently known identifier, including a retained `Retired` record; a released retired record MAY be omitted without permitting identifier reuse.
- **LINK-MED-047** (restatement of LINK-SVC-086): A per-link snapshot is current only while the link's `state` is `Available`.
- **LINK-MED-048**: Per-link snapshot ordering within the `Enumerate-Links` result is implementation-defined and carries no semantics.
- **LINK-MED-049** (specification-suite invariant): v1 defines exactly one link-enumeration primitive, `Enumerate-Links`, and no metrics-aggregator primitive; later Link Specification sections may add mechanism-specific primitives under LINK-SVC-096 and extensions may define non-v1 enumeration mechanisms.
- **LINK-MED-050**: A shim conforms to this specification if and only if it satisfies every applicable `LINK-MED-NNN` implementation requirement.
- **LINK-MED-051**: A conforming shim MUST report a descriptor that satisfies the validation and consistency rules of section 5 for any medium binding it intends to surface.
- **LINK-MED-052** (restatement of LINK-OVR-021): New media are supported by writing new shims; this specification is not revised to enumerate or admit media.
- **LINK-MED-053** (restatement of LINK-OVR-007): An implementation MAY ship shims for any finite subset of media; additional shims are implementation work and do not affect v1 compliance.
- **LINK-MED-054** (restatement of LINK-OVR-024): All per-link state owned by the service or shim, including descriptor, parser, native-unit adaptation, and queued-send state, MUST be resource-bounded and remain correct under adversarial input.
- **LINK-MED-055**: The link service MUST NOT interpret a `started` result from `Shim-Start` as asserting that the medium is up; medium availability is surfaced by `Shim-State-Changed` and mapped by the lifecycle governor.
- **LINK-MED-056**: The link service MUST invoke `Shim-Start` at most once per `link_id`; `Shim-Stop` permanently retires the binding and identifier.
- **LINK-MED-057**: `Shim-Start` for an existing or retired identifier MUST return `rejected-invalid-state` without altering any binding.
- **LINK-MED-058**: After `Shim-Stop`, the shim MUST NOT emit further receive or state-change indications for that `link_id`.
- **LINK-MED-059** (restatement of LINK-SVC-062): Following `Shim-Stop`, the identifier MUST NOT be started or reused during the host run; rediscovery of the same medium uses a new identifier.
- **LINK-MED-060**: A single medium reception MAY yield zero, one, or several `Shim-Receive` indications, depending on how the native reception maps to generic frames.
- **LINK-MED-061** (restatement of LINK-SVC-069 and LINK-SVC-070): A `Shim-Receive` indication carrying a frame that fails section 03 validation MUST be discarded by the link service; the discard MUST NOT affect other receptions or links.
- **LINK-MED-062**: A `kind` of `up` does not by itself cause a `Link-Up` if the reported descriptor fails validation per section 5.
- **LINK-MED-063** (restatement of LINK-SVC-050): The shim MUST NOT silently retry a medium connection or conceal state changes from the link service; any state the shim wishes to surface MUST be surfaced through `Shim-State-Changed`.
- **LINK-MED-064**: A shim MAY emit `down` after it has emitted `up` and MUST NOT depend on knowing whether the governor emitted `Link-Up`.
- **LINK-MED-065**: Lifecycle coalescing without events is limited to wholly pre-availability `up, down` sequences and redundant same-state reports; after exposed availability, accepted `down` and later `up` MUST produce events and a new epoch. Every permitted coalescing remains bounded and observable.
- **LINK-MED-066**: A bare stream without an established native record and recovery convention MUST use the canonical profile defined with section 03.
- **LINK-MED-067**: A native input event that cannot contribute to a valid generic frame within the adapter bounds MUST be discarded without affecting other contexts or links.
- **LINK-MED-068**: A generic frame presented to `Shim-Send` that exceeds the link's reported `maximum_frame_size` MUST be rejected as `rejected-malformed`; fragmentation of Network-layer payloads is performed inside the link service per LINK-SVC-006 and is not a shim responsibility.
- **LINK-MED-069** (restatement of LINK-SVC-020): An implementation MUST NOT infer a capability the descriptor does not report.
- **LINK-MED-070**: A descriptor consists of the core field-set of section 4.2 and the `extensions` collection and nothing else; a descriptor carrying any other component is invalid under section 5.
- **LINK-MED-071**: For `Link-Changed` `change_set`, the extensions collection is the single field `descriptor.extensions`; duplicate `extension_id` entries are permitted and handled implementation-definedly.
- **LINK-MED-072**: An invalid required core field after availability affects availability and MUST be followed by the down barrier and `Link-Down` with `descriptor-invalid`.
- **LINK-MED-073**: Optional values and optional consistency failures MUST be normalised to their defined safe defaults before validity; the authoritative copy stores the normalised descriptor and `Link-Changed` is emitted only if the coherent result differs.
- **LINK-MED-074**: The link service MUST discard any upward indication whose `link_id` is not bound to the emitting shim by an active `Shim-Start`, any indication for a `link_id` after `Shim-Stop` of that binding, any `Shim-State-Changed` indication with an unknown `kind`, and any structurally malformed indication; discards MUST NOT affect other links or receptions and MAY be reported through the LINK-MED-042 diagnostic channel.
- **LINK-MED-075**: The Network layer SHOULD call `Link-Query` for the specific `link_id` when it requires a fresh value after an `Enumerate-Links` snapshot.
- **LINK-MED-076** (restatement of LINK-OVR-006): Conformance to media-specific behaviour that is reported as absent in the validated descriptor is not required.
- **LINK-MED-077** (restatement of LINK-SVC-020 and LINK-SVC-021): The capability descriptor is the sole determinant of which conditional requirements apply.
- **LINK-MED-078** (specification-suite invariant): Sections 06 and 08 MUST define bounded lifecycle-event processing and shed behaviour under resource pressure, including the observability form of the coalescing permitted by LINK-MED-065.
- **LINK-MED-079**: `Shim-Stop` on an `Available` binding MUST follow a down-purpose quiesce barrier; an already equivalent cut-off suffices in a non-`Available` state.
- **LINK-MED-080**: `Shim-Send` MUST return `rejected-link-unavailable` while not up or quiescing, and `unknown-binding` for an unknown or retired identifier.
- **LINK-MED-081**: `Shim-Quiesce` establishes a cut-off for `down` or `size-change` and returns exactly `quiesced`, `failed-timeout`, `unknown-binding`, or `rejected-invalid-state`.
- **LINK-MED-082**: During quiesce new sends are rejected; size-change may resume after the new tuple, while down remains quiescing until `Shim-Stop` and cannot resume.
- **LINK-MED-083**: An adapter profile MUST define the irreversible native-transmission commit boundary; down may permit a pre-cut-off committed frame to complete after the barrier, while size-change MUST wait for committed work to complete.
- **LINK-MED-084**: `quiesced` cancels every uncommitted frame for down or returns every occurrence with exact bytes once for size-change, without coalescing identical frames; the shim retains none.
- **LINK-MED-085**: Before shim `down`, the shim establishes its cut-off; when admitted, the governor closes service admission and classifies service-owned work before `Link-Down`.
- **LINK-MED-086**: A stream with an established native adapter profile MAY use it only when peers share it and it satisfies this section's safety requirements.
- **LINK-MED-087**: Every stream profile MUST define finite parser buffer, progress or dwell, and work bounds, deterministic discard and resynchronisation, and link isolation.
- **LINK-MED-088**: A malformed prefix alone MUST NOT make a link unavailable; a profile unable to resynchronise safely MUST define its reset condition and lifecycle outcome.
- **LINK-MED-089**: Native-unit subdivision state MUST have finite byte, context, work, and lifetime bounds, finite abandonment, exact reconstruction, and no partial-frame exposure.
- **LINK-MED-090**: Every core capability has the directional scope stated in section 4.2; a capability required for correctness in both directions MUST use explicit direction-qualified core fields rather than an extension.
- **LINK-MED-091**: The same ordered descriptor normalisation and validation pipeline applies before first availability and after availability.
- **LINK-MED-092**: Every enumeration record MUST be coherent at one per-link observation point with events ordered around the baseline; an `Available` record authorises `Send`; the collection need not be globally atomic.
- **LINK-MED-093**: A borrowed enumeration view MUST remain immutable for a defined borrow lifetime.
- **LINK-MED-094**: A valid `Shim-Receive` may become `Receive` only during the current `Available` episode under LINK-SVC-091 and LINK-SVC-092.
- **LINK-MED-095**: Before size-reducing `changed`, the shim drains prior old-tuple work; when admitted, the governor establishes the service-side cut-off and classifies concurrent and queued service work before `Link-Changed`.
- **LINK-MED-096**: A binding is `bound-down`, `bound-up`, `quiescing`, or terminal `retired`; successful down quiesce remains `quiescing` until stop, while size-change returns to `bound-up` after `Link-Changed`.
- **LINK-MED-097**: Every operation outside its valid binding state MUST return its defined safe rejection without changing binding, medium, or other-link state; a retired identifier MUST NOT create another binding during the host run.
- **LINK-MED-098**: A down barrier fences every complete or incomplete receive input, adaptation context, callback, byte, native unit, and metadata item from the ending episode so none contributes later.
- **LINK-MED-099**: Upward indications for one binding MUST be delivered in emission order; retained pre-cut-off receives precede the corresponding `down`, while different bindings may interleave.
- **LINK-MED-100**: Every barrier has a finite deadline; timeout returns `failed-timeout`, keeps admission closed, fences uncommitted work, withholds a reduced descriptor, and invokes LINK-SVC-104.
- **LINK-MED-101**: A successful `Shim-Describe` has a binding-local observation point ordered with descriptor-bearing indications, and an older result MUST NOT replace logically newer descriptor state.
- **LINK-MED-102**: `changed` is accepted only during established availability; otherwise it is discarded in isolation, and a pre-availability correction requires another `up`.
- **LINK-MED-103**: Every named core field has at most one logical occurrence; a duplicate core occurrence is a hard structural failure, while duplicate extension identifiers remain permitted.
- **LINK-MED-104**: Descriptor input, depth, traversal, normalisation, comparison, and change-set work have finite per-binding budgets; core or structural exhaustion fails hard, while extension exhaustion deterministically drops excess input.
- **LINK-MED-105**: A timed-out shim-originated barrier MUST close shim admission, fence uncommitted work, and emit non-coalescible `barrier-failed` for the exact binding and purpose; the governor applies LINK-SVC-104 and never publishes a failed reduced tuple.
