# 01. Link Service Model

## 1. Purpose

This section defines the service contract the Link layer exposes upward to the Network Specification, and the contract it expects downward from a shim that adapts a medium. It establishes service primitives, the link lifecycle, multi-link multiplexing, and the limits within which link-layer behaviour is defined.

This section does not redefine any cross-cutting invariant defined in section 00. Where a later section defines a mechanism (framing, fragmentation, addressing, quality, ordering, energy), that section is authoritative for the mechanism; this section is authoritative for the foundational service contract, lifecycle, and multi-link model. A later Link Specification section MAY add a service primitive required to expose the mechanism that section owns, provided the primitive preserves the layering, opacity, lifecycle, and conformance invariants defined here (LINK-SVC-096).

Requirement IDs in this section use the `LINK-SVC-NNN` prefix.

## 2. Upward service contract

### 2.1 Datagram service

The Link layer exposes a datagram service to the Network layer (LINK-SVC-001). A link carries one opaque payload per Send invocation, bounded by the link's reported maximum packet size, and delivers at most one opaque payload per Receive invocation; over time, a link emits zero or more Receive invocations (LINK-SVC-002). The Link layer does not expose a stream-oriented primitive; a medium that is natively stream-oriented is adapted to a datagram link by its shim as defined in section 02 (LINK-SVC-003).

A payload submitted to the Link layer MUST be a contiguous byte sequence (LINK-SVC-004). The Link layer MUST treat the payload as opaque and MUST NOT inspect, parse, or modify its contents (LINK-SVC-005), restating LINK-OVR-002 for this section.

The Link layer MUST NOT fragment a payload submitted by the Network layer across more than one Send invocation; fragmentation, when it occurs, is performed inside the Link layer against the reported MTU and is invisible to the Network layer (LINK-SVC-006). Fragmentation is defined normatively in section 04.

### 2.2 Maximum sizes reported upward

Each link MUST report a current maximum transmission unit (MTU) to the Network layer (LINK-SVC-007). The reported MTU is bounded below by the guaranteed minimum MTU and above by the global maximum frame size; both bounds are defined in section 03 (LINK-SVC-008), restating LINK-OVR-029. The reported MTU MAY change over the life of a link within these bounds; a change is surfaced by the `Link-Changed` primitive (section 4.6).

A link MUST additionally report its maximum packet size, the largest payload the Network layer may submit in one `Send` invocation (LINK-SVC-010). Maximum packet size MAY exceed the MTU because the Link layer may fragment one Network payload into multiple generic frames. Section 03 defines the MTU and generic-frame capacity; section 04 defines maximum packet size using that capacity and its fragmentation and reassembly bounds. The reported maximum packet size MAY change over the life of a link; a change is surfaced by the `Link-Changed` primitive.

The Link layer MUST reject, without sending, any payload whose length exceeds the link's currently reported maximum packet size (LINK-SVC-009). The Link layer MAY reject a payload that is otherwise malformed; behaviour for malformed submit input is defined in section 7.

### 2.3 Delivery semantic

The default delivery semantic of a link is best-effort (LINK-SVC-011). Without further capability reported by the shim, a link:

- MUST NOT be assumed to deliver every submitted payload (LINK-SVC-012).
- MUST NOT be assumed to deliver payloads in submission order (LINK-SVC-013).
- MUST NOT be assumed to suppress duplication (LINK-SVC-014).
- MUST NOT be assumed to perform retransmission (LINK-SVC-015).
- MUST NOT be assumed to provide authenticity, integrity, or confidentiality (LINK-SVC-016), restating LINK-OVR-023.

A shim MAY report, in its capability descriptor, that the medium provides one or more of: reliable delivery, ordered delivery, duplicate suppression, or native retransmission (LINK-SVC-017). Where a capability is reported, is valid per section 02, and is not absent for safety reasons per LINK-OVR-028, the link service MUST provide that property for payloads carried on that link (LINK-SVC-018). All such properties are scoped to a single link and do not transfer across links (LINK-SVC-019).

Whether a capability is present is determined solely by the validated capability descriptor. An implementation MUST NOT infer a capability the descriptor does not report (LINK-SVC-020) and MUST NOT advertise a capability upward that the descriptor does not support (LINK-SVC-021), restating LINK-OVR-028 in this context.

A capability report is an assertion by the shim about the medium, not a measurement. The validation and fail-closed rules of section 3 bound its values, but the Network layer remains free to apply its own end-to-end protections on any link.

Ordering and duplication behaviour, and link-layer retransmission, are defined normatively in sections 07 and 06 respectively; this section states only the contract.

### 2.4 Relationship to the Network layer

The Network Specification builds on the contract defined here and MUST NOT require link behaviour outside this section (LINK-SVC-022), restating LINK-OVR-008 in scope of section 01. In particular, the Network layer MUST NOT assume any delivery semantic stricter than best-effort unless it has read a capability report stating stricter behaviour (LINK-SVC-023).

### 2.5 Identifier boundary between the link service and the Network layer

The link service interface operates only on link-scoped identifiers. Two such identifiers appear in the primitives:

- `link_id`: a host-local opaque handle referring to one of the node's links, as defined in section 6. It is not a link-layer address and does not cross a medium.
- `dest_link_addr` and `src_link_addr`: logical link-layer destinations and sources as defined in section 05. An explicit unicast value includes its incarnation; a source may be absent. These values are meaningful only within one link and epoch and are not network-layer identifiers.

The Link layer MUST NOT resolve network-layer addresses to link-layer addresses (LINK-SVC-074), restating LINK-OVR-002 and LINK-OVR-004 in this context. The link service therefore accepts a link-scoped destination on `Send` and presents a link-scoped source on `Receive`; it cannot accept a network destination.

The Network layer owns the mapping between network-layer addresses and link-scoped identity (LINK-SVC-075). Specifically, the Network layer maintains state mapping a network-layer next hop to `(link_id, epoch, dest_link_addr)` for `Send`, and a present `(link_id, epoch, src_link_addr)` back to a Network peer for `Receive`. An explicit unicast destination or source includes the section 05 incarnation. This state is populated by neighbour observation and by link capability and lifecycle primitives. This requirement does not obligate the Network layer to expose a particular data structure; resolution remains above the Link layer.

The link service exposes the inputs the Network layer uses to maintain that mapping: each link's validated capability descriptor, current epoch, reported MTU and maximum packet size, lifecycle state, and the unauthenticated neighbour-observation primitives defined in section 05 (LINK-SVC-066). The Network layer remains responsible for correlating those facts with Network identity and selecting a next hop (LINK-SVC-096). The link service does not perform link selection; section 6 defines the multi-link model.

## 3. Downward media contract

A link is produced by a shim that adapts a medium. The shim satisfies the shim interface defined normatively in section 02 and reports a capability descriptor describing the medium's properties (LINK-SVC-024), restating LINK-OVR-019 for this section.

A compliant link depends only on what the shim exposes through the shim interface and the capability descriptor. The link service MUST NOT call into the medium's native API except through the shim (LINK-SVC-025). The native API of a medium is out of scope (LINK-OVR-020); this section references capability descriptor fields by name and defers their normative definition to section 02.

Before any capability descriptor value is used to size state or select link behaviour, an implementation MUST validate it against the bounds and consistency rules defined in section 02 (LINK-SVC-026), restating LINK-OVR-027. Where a safety-relevant capability is absent or invalid, the implementation MUST fail closed and behave as if the capability were absent (LINK-SVC-027), restating LINK-OVR-028.

## 4. Service primitives

### 4.1 Form of the primitives

The link service contract is expressed as a set of named logical operations called **primitives**. Each primitive is defined normatively by its **logical inputs**, its **logical outputs**, and the **constraints and ordering rules** that govern how, when, and in what sequence it is invoked; it is not defined by any particular syntactic function shape (LINK-SVC-028).

An implementation conforms to a primitive by exposing an operation that accepts values corresponding to the named logical inputs, produces values corresponding to the named logical outputs, and obeys every stated constraint and ordering rule, regardless of how that operation is expressed in the implementation's language or runtime (LINK-SVC-076). A primitive MAY be realised as a function call returning a value, as a callback or continuation, as a message on a channel or queue, as an actor message, as an event subscription, or by any other mechanism that carries the named inputs and outputs and respects the stated ordering. No representation is mandated or privileged (LINK-SVC-077).

Primitives are listed below by their logical direction. **Upward** primitives are emitted by the link service to the Network layer. **Downward** primitives are invoked by the Network layer on the link service. Every compliant link MUST support all primitives in this section (LINK-SVC-028). The link service implementation is responsible for translating these primitives into and out of the shim interface defined in section 02. In addition to the foundational primitives listed here, the link service provides the `Enumerate-Links` downward primitive defined in section 02. Later Link Specification sections MAY add service primitives only where required to expose their owned mechanisms, as constrained by LINK-SVC-096.

Where a primitive lists an output as one of several alternatives (for example, success or a rejection reason), an implementation MUST surface exactly one of the listed alternatives per invocation (LINK-SVC-078); it MUST NOT silently coalesce alternatives or omit a required output.

Upward primitives for the same `link_id` MUST be delivered to the Network layer in the order they were emitted; upward primitives for different `link_id`s MAY be delivered in any relative order (LINK-SVC-088).

### 4.2 Downward: Send

The `Send` primitive submits an opaque payload for transmission during an identified link availability epoch to an identified link-layer destination (LINK-SVC-029).

**Logical inputs:**

- `link_id`: a handle identifying one of the node's known links per section 6.
- `epoch`: the availability epoch against which the Network layer resolved `dest_link_addr`.
- `payload`: a contiguous opaque byte sequence.
- `dest_link_addr`: exactly one logical implicit, incarnation-qualified unicast, broadcast, or multicast destination as defined in section 05.

**Logical outputs** (exactly one per invocation):

- `accepted`: the payload was accepted for transmission and is accompanied by the non-zero opaque host-local `send_id` defined in section 06.
- `rejected-packet-size-exceeded`: the payload length exceeds the link's currently reported maximum packet size.
- `rejected-malformed`: the inputs are malformed (for example, an unrecognised `link_id`, a zero-length payload, a payload that is not a contiguous byte sequence, or a broadcast destination on a link without broadcast support).
- `rejected-link-unavailable`: the `link_id` is recognised, but the identified link is not in the `Available` lifecycle state or a down-purpose quiesce has closed service admission.
- `rejected-stale-epoch`: the link is `Available`, but the supplied `epoch` is not its current epoch.
- `rejected-direction-unsupported`: the link is `Available`, but its validated descriptor reports a direction that does not support sending.
- `rejected-stale-destination`: the destination is well formed but its observation, incarnation, membership, or native route is not current under section 05.
- `rejected-queue-full`: the link cannot accept the payload because its bounded transmission resources are exhausted. This rejection is transient; the Network layer MAY retry the submission.

**Constraints and ordering:**

- `Send` MUST reject without sending any payload whose length exceeds the link's currently reported maximum packet size (LINK-SVC-009).
- Whether a link-layer broadcast destination is permitted on a given link is governed by the capability descriptor (LINK-SVC-032).
- `Send` MUST NOT block waiting for medium availability beyond a finite, implementation-defined maximum wait interval (LINK-SVC-031).
- `Send` is asynchronous with respect to medium transmission: `accepted` indicates acceptance for transmission, not delivery (LINK-SVC-033).
- Before `accepted`, the service atomically reserves the section 06 identity, immutable ownership, transmission state, and exact terminal-outcome capacity. Failure to reserve those bounded resources is `rejected-queue-full` under LINK-SVC-090.
- If `Send` is invoked on a recognised `link_id` whose lifecycle state is not `Available`, or is ordered after a down-purpose quiesce closed service admission, the output MUST be `rejected-link-unavailable` (LINK-SVC-080).
- If the supplied `epoch` does not equal the current epoch of an `Available` link, the output MUST be `rejected-stale-epoch`, and the payload MUST NOT be queued or transmitted (LINK-SVC-101).
- If the link is `Available` but its validated direction does not support sending, the output MUST be `rejected-direction-unsupported` (LINK-SVC-089).
- Where more than one rejection condition applies, the service MUST select the first applicable result in this order: unrecognised `link_id` as `rejected-malformed`; recognised but non-`Available` lifecycle state or closed down admission; stale epoch; unsupported send direction; malformed payload representation or zero length; structurally malformed or descriptor-unsupported destination; well-formed stale destination; packet size exceeded; queue capacity exhausted (LINK-SVC-090).
- The set of outputs is exactly the eight alternatives above (LINK-SVC-030).

### 4.3 Upward: Receive

The `Receive` primitive delivers a payload that arrived on an identified link from an identified link-layer source (LINK-SVC-034).

**Logical inputs:** none; `Receive` is emitted by the link service.

**Logical outputs** (presented to the Network layer):

- `link_id`: the link on which the payload arrived.
- `epoch`: the current availability epoch of that link.
- `payload`: the opaque byte sequence as received.
- `src_link_addr`: when the received Data frame carried a valid source, its incarnation-qualified unicast Link source as defined in section 05; otherwise absent.
- `recv_meta`: per-receive metadata that the Network layer MAY consult. `recv_meta` MAY be empty and MAY carry shim-reported receive quality hints, such as a signal-strength indication or a receive timestamp. Quality hints are unauthenticated, medium-observable values. The normative contents of `recv_meta` are defined in section 06.

**Constraints and ordering:**

- Correlating a present `(link_id, epoch, src_link_addr)` to a Network peer is the responsibility of the Network layer using its own state per section 2.5; the link service MUST NOT be required to perform that correlation (LINK-SVC-079). An absent source provides no return address or peer-correlation input.
- `Receive` MUST be emitted only while the identified link is `Available`, and its `epoch` MUST equal the epoch of that availability episode (LINK-SVC-091). A reception admitted before the down cut-off MUST either be delivered before `Link-Down` or be discarded through the observable discard mechanism defined in section 06. A reception received before the first `Link-Up` or after the down cut-off MUST be discarded and MUST NOT be retained for a later epoch (LINK-SVC-092).
- A payload MUST be passed up by `Receive` at most once per link-layer reception. If the capability descriptor reports duplicate suppression as absent and the link service does not perform duplicate suppression, a single medium reception MAY surface as multiple `Receive` invocations (LINK-SVC-035).
- Duplicate suppression is defined normatively in section 07.
- Under resource pressure, an implementation MAY discard a valid received frame instead of emitting `Receive` for it. Every such discard MUST be observable to the Network layer, for example through a per-link discard counter or a discard indication carried in a subsequent `recv_meta`; the normative form of the discard signal is defined in section 06 (LINK-SVC-081).

### 4.4 Upward: Link-Up

The `Link-Up` primitive indicates that a link has become available for use (LINK-SVC-036).

**Logical inputs:** none; `Link-Up` is emitted by the link service.

**Logical outputs** (presented to the Network layer):

- `link_id`: the handle of the link that has come up.
- `descriptor`: the validated capability descriptor for the link as defined in section 02; a snapshot of the link-service-owned authoritative copy (section 6.3).
- `epoch`: a generation value identifying the link's current medium binding (LINK-SVC-082).
- `mtu`: the current MTU derived by the link service under section 03.
- `max_packet_size`: the current maximum packet size derived by the link service under section 04 using the framing inputs defined in section 03. A receive-only link reports the section 04 no-send sentinel of zero.

**Constraints and ordering:**

- A link is considered available only after `Link-Up` has been emitted and before any subsequent `Link-Down` for the same `link_id` (LINK-SVC-037).
- The Network layer MAY begin issuing `Send` after either `Link-Up` or an authoritative `Available` baseline returned by `Link-Query` or `Enumerate-Links` for that `link_id` (LINK-SVC-038).
- The descriptor, epoch, MTU, and maximum packet size carried by one `Link-Up` MUST form one internally coherent tuple observed at the event's logical emission point (LINK-SVC-093). A receive-only tuple carries `max_packet_size` equal to zero. A send-capable tuple derives its stable conservative value from the validated capability envelope under sections 04 and 05; the absence of a currently usable destination does not by itself change that value or authorise a `Send`.
- `Link-Up` MUST NOT be emitted for a `link_id` that is currently `Available` (LINK-SVC-049).
- The `epoch` carried by `Link-Up` MUST differ from the epoch of every previous `Link-Up` for the same `link_id`; each `Link-Up` begins a distinct availability episode of the continuing binding (LINK-SVC-082). The epoch space is implementation-defined, but an implementation MUST retire the binding or otherwise fail safely before it would reuse an epoch for that `link_id` during the host run.
- The Network layer MUST treat a changed `epoch` for a `link_id` as invalidating all state associated with the previous epoch, including `(link_id, epoch, dest_link_addr)` and `(link_id, epoch, src_link_addr)` mappings (LINK-SVC-082).

### 4.5 Upward: Link-Down

The `Link-Down` primitive indicates that a link is no longer available for use (LINK-SVC-039).

**Logical inputs:** none; `Link-Down` is emitted by the link service.

**Logical outputs** (presented to the Network layer):

- `link_id`: the handle of the link that has gone down.
- `reason`: an implementation-defined value drawn from the set defined in section 5.4, indicating why the link went down. `reason` is informative; conformance does not depend on its exact value (LINK-SVC-042).

**Constraints and ordering:**

- After `Link-Down`, `Send` on that `link_id` MUST yield `rejected-link-unavailable` until a subsequent `Link-Up` for the same `link_id` (LINK-SVC-040).
- `Link-Down` does not, by itself, release resources owned by open reassembly contexts or retransmission buffers on the link; resource handling on link termination is defined in sections 04 and 06, and release on transition to `Retired` is required by LINK-SVC-085 (LINK-SVC-041).
- `Link-Down` for a `link_id` that is already `Unavailable`, `Unavailable-Pending-Up`, or `Retired` MUST NOT be emitted (LINK-SVC-049).
- Before a service-originated `Link-Down` is emitted, the link service MUST complete the binding-scoped quiesce barrier defined in section 02 (LINK-SVC-094). The cut-off closes service admission and classifies every service-owned and shim-owned frame exactly once. A `Send` ordered after the cut-off is rejected as `rejected-link-unavailable`; service-owned uncommitted work is cancelled; shim-owned uncommitted work is cancelled by the barrier; and no uncommitted frame may begin transmission afterwards. A frame committed before the cut-off MAY complete under the adapter profile. After `Link-Down`, the service MUST complete `Shim-Stop` and transition the identifier to `Retired` (LINK-SVC-102). A shim-originated temporary-loss `down` establishes an equivalent cut-off but MAY transition to `Unavailable-Pending-Up`. The Network layer MUST treat every accepted `Send` without a separately defined successful completion outcome as potentially lost once `Link-Down` is emitted (LINK-SVC-087).
- Section 06 emits every pending exact terminal send outcome and the ending diagnostic snapshot before `Link-Down`. A permitted later physical completion cannot revise an emitted outcome.

### 4.6 Upward: Link-Changed

The `Link-Changed` primitive indicates that a capability or reported service size of an already-up link has changed (LINK-SVC-043).

**Logical inputs:** none; `Link-Changed` is emitted by the link service.

**Logical outputs** (presented to the Network layer):

- `link_id`: the handle of the link whose capability has changed.
- `descriptor`: the validated capability descriptor reflecting the new values, as defined in section 02.
- `mtu`: the new current MTU derived by the link service under section 03.
- `max_packet_size`: the new current maximum packet size derived by the link service under section 04 using the framing inputs defined in section 03, including the zero no-send sentinel where applicable.
- `change_set`: an enumeration of every changed value. Descriptor fields use the `descriptor.` namespace and service values use `service.mtu` and `service.max_packet_size`; the normative descriptor field-set is defined in section 02.

**Constraints and ordering:**

- `Link-Changed` MUST NOT be emitted unless the link is currently `Available` (LINK-SVC-044).
- `Link-Changed` MUST NOT carry a capability value that fails validation against section 02; an invalid reported value MUST fail closed per LINK-SVC-027 and, where the invalid value affects availability, MUST be followed by `Link-Down` (LINK-SVC-045).
- The descriptor, MTU, and maximum packet size carried by one `Link-Changed` MUST form one internally coherent tuple observed at the event's logical emission point (LINK-SVC-093).
- Before a reduced MTU or maximum packet size becomes authoritative through `Link-Changed`, the service and shim MUST establish the size-change barrier defined in section 02 (LINK-SVC-095). Every `Send` is linearised either before the cut-off under the old tuple or after `Link-Changed` under the new tuple. A call ordered after the cut-off is evaluated only after the new tuple is authoritative; its wait remains bounded by LINK-SVC-031, and bounded admission exhaustion returns `rejected-queue-full`. Every old-tuple frame still owned by the service is returned to framing control exactly once. Shim-owned work MUST either complete under the old tuple before the barrier returns or be returned exactly once. Returned best-effort work proceeds to reframing under section 04. Returned work belonging to a reliable fixed frame plan proceeds under section 06 and is either continued byte-identically when the plan and its governing semantics remain valid or cancelled when the transaction terminalises. The service MUST NOT promise to reframe committed work; the barrier waits for it to complete (LINK-SVC-046).
- A change of medium binding is not a capability change. The old binding is retired and any rediscovered binding receives a new `link_id`; it is never surfaced as `Link-Changed` (LINK-SVC-082).

### 4.7 Downward: Link-Query

The `Link-Query` primitive returns the link service's current authoritative state for an identified link (LINK-SVC-084).

**Logical inputs:**

- `link_id`: a handle identifying one of the node's links per section 6.

**Logical outputs** (exactly one alternative per invocation):

- `known`: accompanied by `state` (the link's current lifecycle state per section 5), `descriptor` (the current validated capability descriptor as defined in section 02), `epoch` (the generation value most recently reported by `Link-Up`), `mtu` (the currently reported MTU), and `max_packet_size` (the currently reported maximum packet size). For a known `link_id` that has never validated a descriptor, `descriptor` is `null`/absent and `epoch`, `mtu`, and `max_packet_size` are `unset`. For a known `link_id` that is not currently `Available` but has previously validated a descriptor, the returned values are the most recently validated or reported values and are stale per LINK-SVC-086.
- A `known` result MAY include the same authoritative current section 06 diagnostic snapshot, or the most recent final snapshot marked stale while an unavailable or retired record remains retained. It MUST NOT construct a competing quality baseline.
- `unknown-link`: the `link_id` is not recognised.

**Constraints and ordering:**

- `Link-Query` MUST NOT change link state and MUST NOT have side effects on any link or flow (LINK-SVC-084).
- The returned descriptor and size values are current only while the returned `state` is `Available` (LINK-SVC-086).
- A `known` result MUST be internally coherent: its state, descriptor, epoch, MTU, and maximum packet size represent one logical observation point during the invocation (LINK-SVC-097). For the same consumer, lifecycle events logically preceding that point MUST be delivered before the result becomes observable, and events logically following it MUST be delivered afterwards. An `Available` result establishes the same authority to invoke `Send` as a `Link-Up` event (LINK-SVC-038).

## 5. Link lifecycle

### 5.1 States

Each allocated `link_id` is in exactly one of the following states until its record is released: `Unavailable`, `Available`, `Unavailable-Pending-Up`, or `Retired` (LINK-SVC-047). A `link_id` is `Unavailable` from allocation until its first `Link-Up`. `Retired` is terminal.

- `Unavailable`: the link is not in use. `Send` returns `rejected-link-unavailable`.
- `Available`: the link has emitted `Link-Up` and not yet emitted `Link-Down`. `Send` is permitted.
- `Unavailable-Pending-Up`: the link has emitted `Link-Down` and the same `link_id` is expected to come back; the service MAY keep host-local state, bounded per LINK-SVC-072, but MUST NOT accept `Send` until a subsequent `Link-Up` (LINK-SVC-048).
- `Retired`: the binding has completed `Shim-Stop`, all per-link state has been released, the identifier will never be reused during the current host run, and no further primitive or indication can make it available (LINK-SVC-098). A retired record MAY be removed from query and enumeration state, after which the identifier is unrecognised.

Transitions are caused by `Link-Up`, `Link-Down`, the quiesce and stop operations of section 02, and the implementation's lifecycle governor (section 5.4). A shim-originated temporary medium-loss `down` transitions an `Available` binding to `Unavailable-Pending-Up`. Every service-originated down-purpose quiesce instead completes the barrier, emits `Link-Down` if the link was `Available`, completes `Shim-Stop`, and transitions the identifier to `Retired`. State transitions are summarised:

| From | Event | To |
|------|-------|----|
| Unavailable | Link-Up | Available |
| Available | Shim-originated temporary-loss Link-Down | Unavailable-Pending-Up |
| Available | Service quiesce, Link-Down, Shim-Stop | Retired |
| Unavailable-Pending-Up | Link-Up | Available |
| Unavailable-Pending-Up | Dwell timeout, Shim-Stop | Retired |
| Unavailable | Shim-Stop | Retired |

`Link-Changed` does not appear in the table because it does not change the link's lifecycle state; it updates the validated capability descriptor only (section 4.6).

A lifecycle event of the same type MUST NOT be emitted twice in succession for the same `link_id` without an intervening event of the other type (LINK-SVC-049): no `Link-Up` while already `Available`, and no `Link-Down` while `Unavailable`, `Unavailable-Pending-Up`, or `Retired`. A `Link-Down` MUST NOT be emitted for a link that has never emitted `Link-Up`; the lifecycle governor handles a shim `up, down` sequence coalesced before availability without emitting either service event (LINK-SVC-049).

### 5.2 Intermittent links

A link that reports intermittence in its capability descriptor is intermittent. Intermittent links surface state changes through the lifecycle primitives only; the link service MUST NOT silently retry medium connection or conceal state changes from the Network layer (LINK-SVC-050). Reconnection is not promised by the Link layer and is not performed by the service (LINK-SVC-051). Bounded, faithful, observable coalescing of shim-reported state changes by the lifecycle governor under resource pressure, where permitted by section 02 (LINK-MED-065), is not concealment for the purposes of LINK-SVC-050.

Whether a shim internally attempts to re-establish the medium connection is a shim behaviour, out of scope for this specification; the service surfaces whatever state the shim reports through the lifecycle primitives.

### 5.3 One-way and asymmetric links

A shim MAY report that a medium is one-way in its capability descriptor: send-only, receive-only, or asymmetric (LINK-SVC-052). For a send-only link, the service MUST accept `Send` calls, subject to the rejection outputs defined in section 4.2, and MUST NOT emit `Receive` (LINK-SVC-053). For a receive-only link, the service MUST emit `Receive`, subject to the overload discard permission of LINK-SVC-081, and MUST reject `Send` as `rejected-direction-unsupported` (LINK-SVC-054); the link remains `Available`. An asymmetric link supports both `Send` and `Receive`; each capability field has the normative directional scope defined in section 02, and the constraints of sections 4.2 and 4.3 apply in their respective directions (LINK-SVC-083). A capability required for correctness in both directions MUST use explicit direction-qualified core fields rather than an extension (LINK-SVC-099). Lifecycle primitives apply to one-way and asymmetric links unchanged.

### 5.4 Lifecycle governor and reasons

Each implementation MUST provide a lifecycle governor that maps ordered shim-reported state into lifecycle primitives (LINK-SVC-055). A shim may emit `down` after it has emitted `up`; it does not wait for acknowledgement that the governor emitted `Link-Up`. The governor MUST maintain the latest shim-reported state and MAY coalesce an `up, down` sequence only while no authoritative `Available` episode has been exposed, subject to LINK-MED-065 (LINK-SVC-100). Once availability has been exposed, every accepted `down` MUST produce `Link-Down`, and a later accepted `up` for a continuing temporary-loss binding MUST produce `Link-Up` with a new epoch (LINK-SVC-103). The governor MUST impose a bounded, implementation-defined maximum dwell time in `Unavailable-Pending-Up`; if exceeded, it MUST complete `Shim-Stop`, release all remaining per-link state, and transition the identifier to `Retired` (LINK-SVC-056). State retained while `Unavailable-Pending-Up` MUST remain bounded per LINK-SVC-072. On transition to `Retired`, all remaining per-link state, including open reassembly contexts, neighbour observations, native-route associations, and retransmission buffers referenced by LINK-SVC-041, MUST be released (LINK-SVC-085); sections 04 through 06 define the concrete bounds for that state.

Every quiesce barrier MUST have a finite implementation-defined or adapter-profile-defined deadline (LINK-SVC-104). Failure is reported by `Shim-Quiesce.failed-timeout` or the non-coalescible shim-originated `barrier-failed` indication. The lifecycle governor MUST close service admission, MUST NOT publish a pending reduced tuple, MUST emit `Link-Down` with a barrier-failure reason if availability was exposed, and MUST complete or force `Shim-Stop` and retire the binding. No uncommitted work from the failed barrier may enter another episode. Work already beyond the adapter profile's commit boundary remains subject to that boundary. The failure MUST be observable through the lifecycle reason or implementation diagnostic channel (LINK-SVC-104).

`reason` values carried by `Link-Down` are drawn from an implementation-defined set that MUST include at least the following categories (LINK-SVC-057):

- `medium-loss`: the shim reports the medium connection is lost.
- `descriptor-invalid`: a capability descriptor value failed validation and the link could not fail closed safely.
- `resource`: host-local resources for the link are exhausted.
- `barrier-failure`: a down or size-change ownership barrier failed or exceeded its finite deadline.
- `administrative`: the host or user requested the link be taken down.
- `unknown`: the cause does not fit another category.

The exact membership and encoding of reason values beyond these categories is implementation-defined (LINK-SVC-058).

## 6. Multi-link multiplexing

### 6.1 Link identity

A node MAY host multiple links simultaneously (LINK-SVC-059). Each link is identified by a host-local `link_id` that is unique across every identifier allocated during the current host run (LINK-SVC-060). A `link_id` is an opaque identifier internal to the node; it does not cross a medium, is not a link-layer address, and is not visible to other nodes (LINK-SVC-061).

A `link_id` MUST be stable across `Link-Up` and `Link-Down` cycles while the same binding persists. `Shim-Stop` permanently retires the binding and identifier; the identifier MUST NOT be reused during the current host run, even after its record is removed from query and enumeration state (LINK-SVC-062). Rediscovery of the same medium after retirement MUST allocate a new `link_id`. The identifier space and host-run boundary are implementation-defined, but allocation MUST avoid reuse and MUST fail safely before wraparound. A host process or device lifecycle restart MAY reset the identifier space.

A host-run boundary ends the validity of every host-local `link_id`, epoch, snapshot, and Network mapping from that run (LINK-SVC-062). If a Network consumer survives the restart of another component, that component restart cannot reset the host-run identity space unless the implementation also invalidates all of the consumer's prior local state.

### 6.2 Send across multiple links

When the Network layer issues `Send` on a specific `link_id`, the link service MUST transmit the payload on the identified link only (LINK-SVC-063). The link service MUST NOT silently forward a payload submitted to one link onto another link (LINK-SVC-064). Selecting which link to send on is the responsibility of the Network layer; the link service does not perform link selection (LINK-SVC-065).

### 6.3 Link selection inputs

Although the link service does not select the link, it MUST expose the inputs the Network layer uses to select: each link's validated capability descriptor, its current epoch, its reported current MTU and maximum packet size, and its current lifecycle state (LINK-SVC-066). These inputs are surfaced through the lifecycle primitives (sections 4.4 to 4.6) and through the `Link-Query` primitive (section 4.7); this section adds no further foundational selection primitive (LINK-SVC-067). `Enumerate-Links` is defined in section 02.6, and later sections may add mechanism-specific primitives under LINK-SVC-096.

The authoritative copy of each link's validated capability descriptor is owned by the link service. The descriptors carried by `Link-Up` and `Link-Changed` are coherent snapshots of that copy and the associated service size values at emission time; the Network layer MAY obtain a coherent current tuple for a known `link_id` via `Link-Query` (LINK-SVC-084). Descriptor and size values reported for a `link_id` are current only while that link is `Available` (LINK-SVC-086).

Which media become links is a link-layer concern below the service boundary; which existing link carries a given `Send` is the Network layer's choice. The link service exposes the inputs for that choice but does not act on them.

## 7. Limits and input validation

The link service detects and rejects malformed input at every primitive boundary. Sections 03 through 05 define additional frame-level validation.

Input to `Send` that cannot be accepted, including malformed payloads, an unrecognised or unavailable link, a stale epoch, an unsupported direction, a malformed or unsupported destination, a stale destination, an oversized packet, or exhausted queue capacity, MUST be rejected under the precedence defined by LINK-SVC-090, without transmission and without side effects on any other link or flow (LINK-SVC-068), restating LINK-OVR-014 for the upward boundary.

A `Receive` invocation carrying a payload that fails the link-layer frame validations defined in sections 03 through 05 MUST be discarded and MUST NOT cause the link to become unavailable (LINK-SVC-069). A single malformed reception MUST NOT corrupt state belonging to other receptions or other links (LINK-SVC-070). Discard of valid frames under resource pressure, where permitted, is governed by LINK-SVC-081.

The number of links a node may host simultaneously is implementation-defined; an implementation MAY impose a finite upper bound (LINK-SVC-071). All link-layer state for each link MUST be resource-bounded and MUST remain correct under adversarial input (LINK-SVC-072), restating LINK-OVR-024 for this section.

Every compliant link MUST support at least the guaranteed minimum MTU and MUST NOT exceed the global maximum frame size (LINK-SVC-073), restating LINK-OVR-029 in primitive terms.

## 8. Terminology used in this section

In addition to the terms defined in section 00, this section uses:

**upward**
Towards the Network layer.

**downward**
Towards the shim and medium.

**primitive**
A named operation of the link service contract.

**`link_id`**
A host-local opaque identifier unique across the current host run and never reused during that run.

**epoch**
A generation value identifying an availability episode for one continuing binding; it changes when the binding changes and may change when the binding does not.

**lifecycle governor**
The implementation component that maps ordered shim-reported state into lifecycle primitives.

**`Retired`**
The terminal state after `Shim-Stop`; the identifier cannot return during the host run.

**reason**
An implementation-defined value carried by `Link-Down` indicating why a link went down.

## 9. Requirement summary

The following normative requirements are defined in this section. Entries marked (restatement) restate a cross-cutting invariant from section 00 in the scope of this section; the section 00 statement is authoritative and a conflict resolves in favour of section 00, and satisfying the section 00 statement is sufficient for conformance with the restated requirement. Entries marked (specification-suite invariant) bind the specification suite, including the Network Specification, rather than a running link implementation; see section 4 of section 00. This section is authoritative for the remaining requirements. Requirement IDs provide stable handles between this section and its conformance artefacts, including language-agnostic test vectors and automated conformance tests.

- **LINK-SVC-001**: The Link layer exposes a datagram service to the Network layer.
- **LINK-SVC-002**: A link carries one opaque payload per Send invocation and at most one opaque payload per Receive invocation; over time, a link emits zero or more Receive invocations.
- **LINK-SVC-003**: The Link layer does not expose a stream-oriented primitive; a stream-oriented medium is adapted to a datagram link by its shim per section 02.
- **LINK-SVC-004**: A payload submitted to the Link layer MUST be a contiguous byte sequence.
- **LINK-SVC-005** (restatement of LINK-OVR-002): The Link layer MUST treat the payload as opaque and MUST NOT inspect, parse, or modify its contents.
- **LINK-SVC-006**: The Link layer MUST NOT fragment a payload across more than one Send invocation. Fragmentation is performed inside the Link layer against the reported MTU and is invisible to the Network layer; defined normatively in section 04.
- **LINK-SVC-007**: Each link MUST report a current MTU to the Network layer.
- **LINK-SVC-008**: The reported MTU is bounded below by the guaranteed minimum MTU and above by the global maximum frame size; both defined in section 03.
- **LINK-SVC-009**: The Link layer MUST reject, without sending, any payload whose length exceeds the link's currently reported maximum packet size.
- **LINK-SVC-010**: A link MUST report its maximum packet size, the largest payload the Network layer may submit in one `Send` invocation; it MAY exceed the MTU through Link fragmentation and MAY change over the life of a link, surfaced by `Link-Changed`. Section 03 defines MTU and generic-frame capacity; section 04 defines maximum packet size using that capacity and its fragmentation and reassembly bounds.
- **LINK-SVC-011**: The default delivery semantic of a link is best-effort.
- **LINK-SVC-012**: By default a link MUST NOT be assumed to deliver every submitted payload.
- **LINK-SVC-013**: By default a link MUST NOT be assumed to deliver payloads in submission order.
- **LINK-SVC-014**: By default a link MUST NOT be assumed to suppress duplication.
- **LINK-SVC-015**: By default a link MUST NOT be assumed to perform retransmission.
- **LINK-SVC-016** (restatement of LINK-OVR-023): The Link layer MUST NOT be assumed to provide authenticity, integrity, or confidentiality.
- **LINK-SVC-017**: A shim MAY report in its capability descriptor that the medium provides reliable delivery, ordered delivery, duplicate suppression, or native retransmission.
- **LINK-SVC-018**: Where a capability is reported, is valid per section 02, and is not absent for safety reasons per LINK-OVR-028, the link service MUST provide that property for payloads carried on that link.
- **LINK-SVC-019**: All such properties are scoped to a single link and do not transfer across links.
- **LINK-SVC-020**: An implementation MUST NOT infer a capability the descriptor does not report.
- **LINK-SVC-021**: An implementation MUST NOT advertise a capability upward that the descriptor does not support.
- **LINK-SVC-022** (specification-suite invariant; restatement of LINK-OVR-008): The Network Specification MUST NOT require link behaviour outside this section.
- **LINK-SVC-023** (specification-suite invariant): The Network layer MUST NOT assume any delivery semantic stricter than best-effort unless it has read a capability report stating stricter behaviour.
- **LINK-SVC-024** (restatement of LINK-OVR-019): A link is produced by a shim that satisfies the shim interface defined in section 02 and reports a capability descriptor.
- **LINK-SVC-025**: The link service MUST NOT call into the medium's native API except through the shim.
- **LINK-SVC-026** (restatement of LINK-OVR-027): Before any capability descriptor value is used, an implementation MUST validate it against the bounds and consistency rules defined in section 02.
- **LINK-SVC-027** (restatement of LINK-OVR-028): Where a safety-relevant capability is absent or invalid, the implementation MUST fail closed and behave as if the capability were absent.
- **LINK-SVC-028**: Every compliant link MUST support all primitives in section 4. Each primitive is defined normatively by its logical inputs, logical outputs, and the constraints and ordering rules that govern its invocation, not by any particular syntactic function shape.
- **LINK-SVC-029**: The `Send` primitive submits an opaque payload for transmission on an identified link and availability epoch to an identified link-layer destination.
- **LINK-SVC-030**: The set of `Send` outputs is exactly `accepted` with section 06 `send_id`, `rejected-packet-size-exceeded`, `rejected-malformed`, `rejected-link-unavailable`, `rejected-stale-epoch`, `rejected-direction-unsupported`, `rejected-stale-destination`, and `rejected-queue-full`.
- **LINK-SVC-031**: `Send` MUST NOT block waiting for medium availability beyond a finite, implementation-defined maximum wait interval.
- **LINK-SVC-032**: Whether a link-layer broadcast destination is permitted on a given link is governed by the capability descriptor.
- **LINK-SVC-033**: `Send` is asynchronous with respect to medium transmission; `accepted` indicates acceptance for transmission, not delivery.
- **LINK-SVC-034**: `Receive` delivers a payload with link, epoch, optional incarnation-qualified source, and section 06-owned receive metadata; anonymous source is absent.
- **LINK-SVC-035**: A payload MUST be passed up by `Receive` at most once per link-layer reception; if the capability descriptor reports duplicate suppression as absent and the link service performs no suppression, a single medium reception MAY surface as multiple `Receive` invocations, subject to the duplicate suppression rules in section 07.
- **LINK-SVC-036**: The `Link-Up` primitive indicates a link has become available. Its logical outputs are `link_id`, `descriptor`, `epoch`, `mtu`, and `max_packet_size`.
- **LINK-SVC-037**: A link is available only after `Link-Up` and before any subsequent `Link-Down` for the same `link_id`.
- **LINK-SVC-038**: The Network layer MAY begin issuing `Send` after `Link-Up` or after an authoritative `Available` baseline returned by `Link-Query` or `Enumerate-Links`.
- **LINK-SVC-039**: The `Link-Down` primitive indicates a link is no longer available. Its logical outputs are `link_id` and `reason`.
- **LINK-SVC-040**: After `Link-Down`, `Send` on that `link_id` MUST yield `rejected-link-unavailable` until a subsequent `Link-Up`.
- **LINK-SVC-041**: `Link-Down` does not by itself release resources owned by reassembly contexts or retransmission buffers; resource handling is defined in sections 04 and 06, and release on transition to `Retired` is required by LINK-SVC-085.
- **LINK-SVC-042**: `reason` is informative; conformance does not depend on its exact value.
- **LINK-SVC-043**: The `Link-Changed` primitive indicates that a capability or service size of an already-up link has changed. Its logical outputs are `link_id`, `descriptor`, `mtu`, `max_packet_size`, and `change_set`; `change_set` MUST enumerate every changed descriptor field or service size using the defined namespaces.
- **LINK-SVC-044**: `Link-Changed` MUST NOT be emitted unless the link is currently `Available`.
- **LINK-SVC-045**: `Link-Changed` MUST NOT carry a descriptor value that fails validation; invalid optional values are normalised safely, and an invalid value that affects availability MUST be followed by `Link-Down`.
- **LINK-SVC-046**: Before a reduced size becomes authoritative, sends are ordered against a barrier; committed old-tuple work completes before it returns, while every uncommitted frame is returned exactly once for best-effort reframing or reliable fixed-plan handling before `Link-Changed`.
- **LINK-SVC-047**: Each allocated `link_id` is in exactly one of `Unavailable`, `Available`, `Unavailable-Pending-Up`, or terminal `Retired` until its record is released; it begins in `Unavailable`.
- **LINK-SVC-048**: In `Unavailable-Pending-Up`, the service MAY keep host-local state, bounded per LINK-SVC-072, but MUST NOT accept `Send` until a subsequent `Link-Up`.
- **LINK-SVC-049**: A lifecycle event of the same type MUST NOT be emitted twice in succession for the same `link_id` without an intervening event of the other type: no `Link-Up` while `Available`, no `Link-Down` while `Unavailable`, `Unavailable-Pending-Up`, or `Retired`, and no `Link-Down` for a link that has never emitted `Link-Up`; a coalesced pre-availability `up, down` sequence emits neither service event.
- **LINK-SVC-050**: Intermittent links surface state changes through the lifecycle primitives only; the link service MUST NOT silently retry medium connection or conceal state changes; bounded, faithful, observable coalescing of shim-reported state changes under resource pressure, where permitted by section 02 (LINK-MED-065), is not concealment.
- **LINK-SVC-051**: Reconnection is not promised by the Link layer and is not performed by the service.
- **LINK-SVC-052**: A shim MAY report that a medium is one-way: send-only, receive-only, or asymmetric.
- **LINK-SVC-053**: For a send-only link, the service MUST accept `Send` calls, subject to the rejection outputs of section 4.2, and MUST NOT emit `Receive`.
- **LINK-SVC-054**: For a receive-only link, the service MUST emit `Receive`, subject to the overload discard permission of LINK-SVC-081, and MUST reject `Send` as `rejected-direction-unsupported` while the link remains `Available`.
- **LINK-SVC-055**: Each implementation MUST provide a lifecycle governor that maps ordered shim-reported state into lifecycle primitives.
- **LINK-SVC-056**: The lifecycle governor MUST impose a bounded, implementation-defined maximum dwell time in `Unavailable-Pending-Up`; if exceeded, it MUST complete `Shim-Stop`, release all per-link state, and transition the identifier to `Retired`.
- **LINK-SVC-057**: `reason` values MUST include at least `medium-loss`, `descriptor-invalid`, `resource`, `barrier-failure`, `administrative`, and `unknown`.
- **LINK-SVC-058**: The exact membership and encoding of reason values beyond the required categories is implementation-defined.
- **LINK-SVC-059**: A node MAY host multiple links simultaneously.
- **LINK-SVC-060**: Each link is identified by a host-local `link_id` unique across every identifier allocated during the current host run.
- **LINK-SVC-061**: A `link_id` is opaque and host-local; it is not a link-layer address and is not visible to other nodes.
- **LINK-SVC-062**: A `link_id` is stable while its binding persists. `Shim-Stop` permanently retires the binding and identifier; the identifier MUST NOT be reused during the current host run, and rediscovery of the same medium MUST allocate a new identifier.
- **LINK-SVC-063**: `Send` on a specific `link_id` MUST be transmitted on the identified link only.
- **LINK-SVC-064**: The link service MUST NOT silently forward a payload submitted to one link onto another link.
- **LINK-SVC-065**: Selecting which link to send on is the responsibility of the Network layer; the link service does not perform link selection.
- **LINK-SVC-066**: The service exposes descriptor, epoch, MTU, maximum packet size, lifecycle state, and section 05 neighbour observations through their owning primitives.
- **LINK-SVC-067**: Beyond `Link-Query`, section 6.3 adds no further foundational selection primitive; `Enumerate-Links` is defined in section 02.6 and mechanism-specific primitives may be added under LINK-SVC-096.
- **LINK-SVC-068**: `Send` input that cannot be accepted MUST be rejected under LINK-SVC-090 without transmission and without side effects on other links or flows.
- **LINK-SVC-069**: A `Receive` carrying a payload that fails sections 03 through 05 frame validation is discarded and does not make the link unavailable.
- **LINK-SVC-070**: A single malformed reception MUST NOT corrupt state belonging to other receptions or other links.
- **LINK-SVC-071**: The number of links a node may host simultaneously is implementation-defined; an implementation MAY impose a finite upper bound.
- **LINK-SVC-072** (restatement of LINK-OVR-024): All link-layer state for each link MUST be resource-bounded and MUST remain correct under adversarial input.
- **LINK-SVC-073** (restatement of LINK-OVR-029 in primitive terms): Every compliant link MUST support at least the guaranteed minimum MTU and MUST NOT exceed the global maximum frame size.
- **LINK-SVC-074** (restatement of LINK-OVR-002 and LINK-OVR-004): The Link layer MUST NOT resolve network-layer addresses to link-layer addresses. The link service accepts a link-scoped destination on `Send` and presents a link-scoped source on `Receive`; it cannot accept a network destination.
- **LINK-SVC-075**: The Network layer owns mappings between Network addresses and incarnation-qualified `(link_id, epoch, link_addr)` values populated by neighbour observation and lifecycle state; resolution remains above Link.
- **LINK-SVC-076**: An implementation conforms to a primitive by exposing an operation that accepts values corresponding to the named logical inputs, produces values corresponding to the named logical outputs, and obeys every stated constraint and ordering rule, regardless of how that operation is expressed in the implementation's language or runtime.
- **LINK-SVC-077**: A primitive MAY be realised as a function call returning a value, as a callback or continuation, as a message on a channel or queue, as an actor message, as an event subscription, or by any other mechanism that carries the named inputs and outputs and respects the stated ordering. No representation is mandated or privileged.
- **LINK-SVC-078**: Where a primitive lists an output as one of several alternatives, an implementation MUST surface exactly one of the listed alternatives per invocation and MUST NOT silently coalesce alternatives or omit a required output.
- **LINK-SVC-079**: Correlating a present `(link_id, epoch, src_link_addr)` to a Network peer is Network-owned; an absent source supplies no correlation or return input.
- **LINK-SVC-080**: `Send` on a recognised non-`Available` link or after down-purpose quiesce closed admission MUST return `rejected-link-unavailable`.
- **LINK-SVC-081**: Under resource pressure, an implementation MAY discard a valid received frame instead of emitting `Receive` for it; every such discard MUST be observable to the Network layer. The normative form of the discard signal is defined in section 06.
- **LINK-SVC-082**: Every `Link-Up` for a `link_id` carries a previously unused epoch for that host run and identifies a distinct availability episode; Network invalidates prior-epoch state, and the implementation retires or fails safely before epoch reuse. A different binding receives a new `link_id`.
- **LINK-SVC-083**: An asymmetric link supports both `Send` and `Receive`; each descriptor field has a normative directional scope, and the service constraints apply in their respective directions.
- **LINK-SVC-084**: The link service MUST provide a side-effect-free `Link-Query` primitive returning an internally coherent lifecycle state, validated descriptor, epoch, MTU, and maximum packet size tuple for a recognised `link_id`, or `unknown-link` for an unrecognised one.
- **LINK-SVC-085**: Retirement releases all per-link reassembly, neighbour, native-route, retransmission, and other state; pending-up state remains bounded.
- **LINK-SVC-086**: Descriptor and size values reported for a `link_id` are current only while that link is `Available`.
- **LINK-SVC-087**: At the down cut-off, uncommitted work MUST be cancelled or returned to service control and MUST NOT begin transmission afterwards; a frame irreversibly committed before the cut-off MAY complete under the adapter profile. The Network layer MUST treat every accepted `Send` without a defined successful completion outcome as potentially lost once `Link-Down` is emitted.
- **LINK-SVC-088**: Upward primitives for the same `link_id` MUST be delivered to the Network layer in the order they were emitted; upward primitives for different `link_id`s MAY be delivered in any relative order.
- **LINK-SVC-089**: If an `Available` link's validated direction does not support sending, `Send` MUST return `rejected-direction-unsupported`.
- **LINK-SVC-090**: Overlapping `Send` rejection conditions are ordered: unrecognised identifier, unavailable lifecycle or closed admission, stale epoch, unsupported direction, malformed or zero-length payload, malformed or unsupported destination, stale destination, packet size, then queue capacity.
- **LINK-SVC-091**: `Receive` MUST be emitted only while the link is `Available` and MUST carry the epoch of that availability episode.
- **LINK-SVC-092**: A reception admitted before the down cut-off MUST be delivered before `Link-Down` or discarded observably; pre-up and post-cut-off receptions MUST be discarded and MUST NOT cross into a later epoch.
- **LINK-SVC-093**: The descriptor, epoch where applicable, MTU, and maximum packet size carried by one lifecycle event MUST form one internally coherent tuple at its logical emission point, including a zero no-send sentinel for receive-only links and the section 04/05 empty-destination case.
- **LINK-SVC-094**: Before a service-originated `Link-Down`, one cut-off closes service admission and classifies all service-owned and shim-owned work; after the event the service stops and retires the binding. Shim-originated temporary loss provides an equivalent cut-off but may retain the binding pending recovery.
- **LINK-SVC-095**: Before a reduced tuple becomes authoritative, every send is linearised under the old or new tuple, all service-owned work is classified, and shim-owned work completes or returns; bounded admission exhaustion may return `rejected-queue-full`.
- **LINK-SVC-096**: Later Link Specification sections MAY add service primitives required to expose mechanisms they own, provided those primitives preserve this section's layering, opacity, lifecycle, and conformance invariants; the Link layer exposes link-local facts while the Network layer owns identity correlation and next-hop selection.
- **LINK-SVC-097**: Each `Link-Query` result MUST represent one coherent per-link observation point, with lifecycle events ordered around that baseline; an `Available` result authorises `Send` as `Link-Up` does.
- **LINK-SVC-098**: `Retired` is terminal: `Shim-Stop` has completed, all per-link state is released, the identifier cannot be reused during the host run, and no later event can make it available.
- **LINK-SVC-099**: A capability required for correctness in both directions MUST use explicit direction-qualified core fields and MUST NOT be relegated to an extension.
- **LINK-SVC-100**: The governor maintains latest ordered shim state and may coalesce an `up, down` sequence only before any authoritative availability is exposed and under LINK-MED-065.
- **LINK-SVC-101**: `Send` MUST carry the expected epoch and MUST return `rejected-stale-epoch` without queueing or transmitting when it does not equal the current epoch of an `Available` link.
- **LINK-SVC-102**: Every service-originated down-purpose quiesce proceeds through `Link-Down`, `Shim-Stop`, and terminal `Retired`; only shim-originated temporary medium loss may enter `Unavailable-Pending-Up`.
- **LINK-SVC-103**: Once availability is exposed, an accepted `down` MUST produce `Link-Down` and a later accepted `up` for the continuing binding MUST produce `Link-Up` with a new epoch.
- **LINK-SVC-104**: Every barrier has a finite deadline; `failed-timeout` or non-coalescible `barrier-failed` reports failure, which closes admission, withholds a reduced tuple, observably forces down and retirement, and fences uncommitted work.
