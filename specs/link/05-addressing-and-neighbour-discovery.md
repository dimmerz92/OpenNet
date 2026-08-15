# 05. Addressing and Neighbour Discovery

## 1. Purpose and boundary

This section defines link-scoped addressing, incarnation-qualified unicast destinations, native-route adaptation, neighbour observation, and the optional active discovery mechanism used by the Link layer (LINK-ADR-001). It defines only local-hop information. Network addressing, routing, next-hop selection, end-to-end identity, and absolute reply identity remain outside the Link layer.

A Link address, incarnation, neighbour observation, or native route MUST NOT embed, assert, or be interpreted as a Network address, upper-layer identity, long-term device identity, authenticated principal, or proof of reachability (LINK-ADR-002). Any absolute source or reply identity belongs inside the opaque Network packet and may be protected there by the Network or a higher layer.

Requirement IDs in this section use the `LINK-ADR-NNN` prefix.

## 2. Address model and wire fields

### 2.1 Logical address

A Link address is an explicitly typed opaque byte string meaningful only within one link and availability epoch (LINK-ADR-003). Equality compares the address kind, value length, and every value byte. An address on another `link_id` or epoch is a different logical address even when its encoded bytes are equal.

The logical address kinds are `unicast`, `broadcast`, `multicast`, and `implicit` (LINK-ADR-004). Unicast identifies one current local claim subject to the incarnation rules below. Broadcast addresses every eligible receiver on the local medium. Multicast identifies an externally configured or adapter-supplied local delivery set. Implicit denotes the peer inherent in a binding and has no wire encoding.

When an address is encoded in a TLV, its value begins with one address-kind octet followed by the opaque address value (LINK-ADR-005):

| Kind octet | Meaning | Opaque value length |
|---:|---|---:|
| `0x00` | invalid | none |
| `0x01` | unicast | 1 to 32 bytes |
| `0x02` | broadcast | 0 bytes |
| `0x03` | multicast | 1 to 32 bytes |

`0x04` through `0xFF` are invalid in version 1. The logical `implicit` kind is represented by omission and therefore has no kind octet. Address bytes are opaque and carry no numeric byte order.

### 2.2 TLV allocation

Version 1 allocates these generic-frame TLV types (LINK-ADR-006):

| Type | Name | Allowed class | Value length |
|---:|---|---|---:|
| `0x06` | Destination Link Address | Data or defined Control | 1 to 33 |
| `0x07` | Destination Incarnation | Data or defined Control | 16 |
| `0x08` | Source Link Address | Data or defined Control | 2 to 33 |
| `0x09` | Sender Incarnation | Data or defined Control | 16 |
| `0x0A` | Advertisement Validity | Neighbour Advertisement Control only | 4 |

These assignments occupy the section 03 version 1 namespace and retain its strict numeric ordering rule. Later sections MUST NOT assign another meaning to types `0x06` through `0x0A`.

Destination Link Address MUST encode exactly one valid kind and length from section 2.1 (LINK-ADR-007). A unicast or multicast value is compared as an opaque exact byte sequence. Broadcast has no opaque value. The implicit destination is never encoded as a Destination Link Address TLV.

Source Link Address MUST encode unicast with 1 through 32 opaque value bytes (LINK-ADR-008). Broadcast, multicast, an empty unicast value, and every unrecognised kind are forbidden as a source.

Destination Incarnation and Sender Incarnation are opaque 16-byte values (LINK-ADR-009). They have no numeric ordering or globally assigned meaning. Equality is exact byte equality. An all-zero incarnation is invalid, preventing an absent or zero-initialised value from becoming a valid claim accidentally.

Advertisement Validity is an unsigned 32-bit big-endian duration in seconds and MUST be non-zero (LINK-ADR-010). It is a relative lifetime observed from admission of the valid advertisement, not a wall-clock timestamp.

### 2.3 Field combinations

An explicit unicast destination MUST contain Destination Link Address and Destination Incarnation exactly once; broadcast and multicast MUST contain Destination Link Address and MUST NOT contain Destination Incarnation (LINK-ADR-011). Destination Incarnation without a unicast Destination Link Address is malformed.

On an `implicit-peer` binding, Data frames MUST omit Destination Link Address and Destination Incarnation (LINK-ADR-012). Their absence on any other binding is malformed unless the definition of a Control message explicitly permits it.

Every transmitted Data frame MUST use exactly one destination form: explicit unicast, broadcast, multicast, or binding-implicit (LINK-ADR-013). Broadcast and multicast transmission additionally require their section 02 send-direction capability. A sender MUST NOT encode more than one form.

Source Link Address is optional on Data frames (LINK-ADR-014). When present, Sender Incarnation MUST also be present. Sender Incarnation MAY be present without Source Link Address only where another section requires sender separation without a routable return address.

A fragmented Data frame MUST carry Sender Incarnation unless the adapter profile supplies an equally stable binding-specific peer incarnation for an implicit single-peer binding (LINK-ADR-015). A source-less sender MAY use a fresh packet-scoped incarnation. Unfragmented anonymous Data may omit both source fields.

Addressing TLVs on a Control frame are permitted only as specified by that Control Type (LINK-ADR-016). A Control Type definition must state its exact destination form, source requirements, incarnation requirements, body syntax, and permitted additional TLVs.

An invalid kind, length, combination, class, zero incarnation, duplicated field, or forbidden field is `addressing-malformed` and MUST be discarded before neighbour, route, fragmentation, or reassembly allocation (LINK-ADR-017). The discard affects no unrelated frame, observation, destination, or link.

The maximum mandatory offset-zero fragmented Data header under this section is 133 bytes: 3 fixed bytes, 24 fragmentation-TLV bytes, two maximum address TLVs, and two incarnation TLVs (LINK-ADR-018). It therefore leaves at least 123 body bytes at MTU 256 before optional padding. Optional padding MUST NOT reduce required representability.

## 3. Neighbour modes and local claims

### 3.1 Capability mode

The section 02 capability descriptor contains exactly one required `neighbour_mode` value from `none`, `implicit-peer`, `externally-supplied`, or `active-advertisement` (LINK-ADR-019). The mode describes proactive acquisition of send destinations; bounded passive observation under section 6 remains separate.

`none` provides no proactive unicast destination (LINK-ADR-020). It is valid on a receive-only binding and on a send-capable binding that uses only valid broadcast or configured multicast destinations. It does not make an explicit unicast address valid.

`implicit-peer` means the binding itself identifies at most one transmission peer and no encoded destination or native-route token is required (LINK-ADR-021). The adapter profile MUST provide a peer-incarnation rule that changes before remote Packet Identifier allocation state can repeat or be lost, or require persistent non-repeating allocation state across such restart.

`externally-supplied` means a bounded local configuration or adapter-owned source supplies each usable unicast `(address, incarnation, native_route)` entry (LINK-ADR-022). The source and removal mechanism are implementation-defined, but a conformance harness MUST be able to add, replace, remove, and exhaust entries without using Network identity as Link identity.

`active-advertisement` uses the Control message in section 5 and is valid only on a send-and-receive-capable binding whose validated descriptor reports `broadcast_support` (LINK-ADR-023). The adapter profile MUST support a native route observation on advertisement reception and a bounded broadcast route on send.

A send-only binding MUST use `none`, `implicit-peer`, or `externally-supplied`; a receive-only binding MUST use `none` (LINK-ADR-024). An invalid combination is a required-field descriptor consistency failure under section 02. Asymmetric and bidirectional bindings may use any mode whose other conditions are satisfied.

### 3.2 Local claims and allocation

The same local `(Link address, Sender Incarnation)` pair MUST NOT be issued to two claims on one link and epoch and MUST NOT be reissued during the current host run (LINK-ADR-025). Link addresses require no central allocator and need not be globally unique.

Before incarnation allocation state could wrap, repeat, or be lost, the implementation MUST establish a new non-zero incarnation or fail safely without issuing a claim (LINK-ADR-026). Restart without preserved non-repeating state requires a new incarnation. Reuse of an opaque address value requires a new incarnation.

Address and incarnation allocation MAY use secure randomness, persistent counters, adapter allocation, collision-checked selection, or another bounded method satisfying the observable invariants (LINK-ADR-027). Before issuing an active claim, the implementation MUST check the candidate against current local claims and current explicit observations, retry within a finite exposed work bound, or fail safely without issue. If an exact collision becomes current after issue, the local claim becomes unusable for new transmission and MUST rotate safely or fail closed within bounded work. This check does not authenticate either claimant. Where the platform exposes a suitable cryptographic entropy source, active-discovery allocation MUST support unpredictable address and incarnation generation. Conformance tests non-reuse, collision handling before and after issue, bounded retry, rotation, and safe exhaustion, not statistical randomness.

An active-discovery implementation MUST support controlled rotation of its local address and incarnation under documented local or higher-layer policy (LINK-ADR-028). Version 1 defines no universal rotation interval. The policy and its externally observable cut-off behaviour MUST be available to a conformance harness.

Rotation creates a new claim and MUST NOT assert continuity with the old address or incarnation (LINK-ADR-029). The old claim is invalidated through section 8 before the new claim is used. A node SHOULD avoid intentionally retiring a claim before its latest advertised validity expires unless it also invalidates the binding or accepts that peers may retain a stale observation.

## 4. Native-route shim boundary

A `native_route` is an opaque, bounded, adapter-owned token identifying how the shim attempts one native transmission or reports one native origin (LINK-ADR-030). It is scoped to one `link_id` and availability epoch, is not a Link address, and MUST NOT be exposed to the Network layer or interpreted as identity.

`Shim-Receive` supplies the reconstructed generic frame with the observed `native_route` when the medium provides one (LINK-ADR-031). The token MAY be absent for an implicit binding or where no reply route is available. The shim validates only its token representation and bounds; the Link service validates and associates Link addressing.

`Shim-Send` supplies the complete generic frame with the `native_route` selected by the Link service and, for a source-bearing frame, an opaque `source_claim_generation` identifying the exact local claim generation used by that frame (LINK-ADR-032). The route token MAY be absent only for a binding whose adapter profile defines an implicit route; the claim token is absent for source-less frames. A stale, foreign-binding, malformed, or oversized token is rejected as `rejected-malformed` without transmission. Neither token is a wire field or identity, and the shim need not parse the frame to use it.

Each adapter profile MUST define finite route-token and source-claim-generation-token size, lifetime, equality, validation, mapping, commitment, selective cut-off, returned-work, and invalidation rules (LINK-ADR-033). Simultaneously usable explicit destinations MUST have independently invalidatable route-token values even when they map to the same native endpoint. A route token from another link or epoch MUST NOT select a route, and a claim token from another claim generation, link, or epoch MUST NOT select source ownership. The implementation MUST release token state when its observation, claim, destination, epoch, or binding ends.

Every uncommitted frame returned by a size-change barrier retains its exact associated `native_route`; a down barrier cancels both together (LINK-ADR-034). Reframing may select a new current token only after the destination remains valid on the new side of the cut-off. Returned route metadata is bounded and preserves one-to-one ownership with its frame.

The shim acts on `native_route`, not on Link-address TLVs (LINK-ADR-035). It is not required to parse Link TLVs, Control Types, fragmentation fields, or payload contents. Full generic-frame semantic validation remains owned by the Link service.

## 5. Active neighbour advertisement

### 5.1 Wire form

Version 1 allocates Control Type `0x0001` as Neighbour Advertisement (LINK-ADR-036). It contains exactly Control Type, broadcast Destination Link Address, unicast Source Link Address, Sender Incarnation, and Advertisement Validity TLVs; it forbids Destination Incarnation and every other section 04 or 05 TLV, and its body is empty. Its maximum header is 69 bytes.

An advertisement exposes only an ephemeral local Link source, its current non-zero incarnation, and a validity duration (LINK-ADR-037). It MUST NOT carry a Network address, device name, vendor identifier, public key, long-term identifier, capability claim, or continuity proof.

An `active-advertisement` adapter profile MUST define a finite `maximum_renewal_opportunity_interval` covering its maximum advertisement interval, jitter, local scheduling, native admission, and transmission allowance while the binding is `Available` (LINK-ADR-038). The transmitted Advertisement Validity, converted to seconds, MUST be strictly greater than that bound because expiry wins at the exact deadline. The sender guarantees one best-effort transmission opportunity within the bound, not reception or uninterrupted observation. A profile unable to satisfy its envelope MUST increase validity, delay active availability, or fail safely.

### 5.2 Receive processing

After complete frame and addressing validation, the effective advertisement lifetime is the smaller of the advertised duration and the receiver's configured finite local maximum (LINK-ADR-039). The local maximum and resulting expiry are exposed to a conformance harness. Admission at a logical observation time establishes the expiry; an exact refresh may replace it but does not affect another entry. Receiver clamping is an independent defensive policy and MAY intentionally produce gaps despite a conforming sender's renewal opportunities.

Version 1 defines no neighbour solicitation, discovery response, or withdrawal message (LINK-ADR-040). Expiry and lifecycle invalidation remove observations. An unknown Control Type is handled under section 03 and cannot be treated as discovery.

## 6. Neighbour observations

### 6.1 Observation identity and sources

An explicit neighbour observation is keyed by `(link_id, epoch, Link address, incarnation, native_route)` (LINK-ADR-041). A validated exact active advertisement refreshes only the exact entry. It allocates no duplicate address bytes or route token where the implementation can safely share an immutable representation.

Two observations with the same address and incarnation but different native routes make that destination ambiguous and unusable for new `Send` calls until only one current route remains (LINK-ADR-042). Different incarnations are distinct destinations even when their address bytes match. An exact address-incarnation collision from different physical senders cannot be resolved as identity and remains an unauthenticated local denial-of-service condition.

On a send-and-receive-capable binding, a valid received Data frame carrying Source Link Address and Sender Incarnation with a valid observed native route MAY create or refresh an exact passive observation (LINK-ADR-043). Mutation occurs only after all applicable section 03 structure, section 05 addressing and local-destination checks, and section 04 fragmentation field, range, and semantic validation have passed, but MAY precede reassembly resource retention because route validity and packet-buffer availability are separate facts. Its lifetime is a finite locally configured duration exposed to conformance. A source-less frame creates none. A receive-only binding may report the source upward but MUST NOT create a usable send destination.

An externally supplied observation uses the same key, resource, ambiguity, event, and destination-cut-off rules (LINK-ADR-044). It has either a finite expiry or an explicit epoch-bounded removal condition. External provenance does not make it a Network identity or protocol authentication result.

An implicit peer is represented by the binding and current peer incarnation rather than an explicit neighbour-table entry (LINK-ADR-045). It becomes invalid at the binding or epoch cut-off and cannot be carried into a later epoch.

### 6.2 Upward and query primitives

Public neighbour state is aggregated by `(link_id, epoch, Link address, incarnation)` and is either `usable` when exactly one current internal route exists or `ambiguous` when more than one exists. `Neighbour-Observed` is an upward primitive carrying that aggregate identity, state, aggregate expiry or `epoch-bound`, and the bounded set of contributing source modes from `active-advertisement`, `passive-data`, and `externally-supplied` (LINK-ADR-046). Aggregate expiry is the latest current finite expiry among its internal observations, or `epoch-bound` if any contributing observation is epoch-bounded. It reports unauthenticated aggregate state only and does not expose `native_route` or assert identity, liveness, bidirectional reachability, continuity, or Network association.

`Neighbour-Changed` carries the complete updated public aggregate record, including identity, `usable` or `ambiguous` state, aggregate expiry or `epoch-bound`, and contributing source modes, whenever any of those public fields changes while the aggregate remains current. `Neighbour-Expired` carries the aggregate identity and exactly one reason from `lifetime-expired`, `resource-evicted`, `externally-removed`, `route-invalidated`, `claim-replaced`, or `epoch-ended` (LINK-ADR-047). It is emitted only when the final internal observation for a surfaced aggregate ceases to be current. Exact reason representation is language-neutral.

`Enumerate-Neighbours` is a downward primitive taking `link_id` and `epoch` and returning exactly one of `current`, `rejected-link-unavailable`, `rejected-stale-epoch`, or `rejected-malformed` (LINK-ADR-048). An unrecognised or released retired `link_id` yields `rejected-malformed`; a recognised link not currently `Available`, including a retained `Retired` record, yields `rejected-link-unavailable`; a wrong epoch on an available link yields `rejected-stale-epoch`; any remaining malformed primitive representation yields `rejected-malformed`; otherwise the result is `current`. This is the precedence for compound conditions, every rejection has no side effects, and structural bounds needed for safe comparison may be checked first without changing the observable result. `current` contains a bounded snapshot of every public field of every current aggregate but excludes native routes. The snapshot is authoritative at one logical observation point and has no side effects.

Repeated exact observations MAY be coalesced, but event delivery and enumeration MUST remain mutually ordered for the same consumer and link (LINK-ADR-049). Events logically before a snapshot become observable before it; later events become observable afterwards. While bounded event capacity permits, coalescing MUST NOT conceal first appearance, final expiry, ambiguity transition, or destination invalidation. If exact incremental history cannot be retained, the service sets at most one sticky `Neighbour-Baseline-Invalidated` state per consumer and link, may shed superseded detail within bounds, and requires a successful `Enumerate-Neighbours` authoritative snapshot at a defined observation point to clear it.

### 6.3 Expiry

At a logical observation time equal to or later than an observation deadline, expiry is processed before advertisement refresh, passive refresh, enumeration, or destination admission (LINK-ADR-050). Timer resolution is implementation-defined, but the exposed deadline and boundary outcome are reproducible. Refresh never crosses an epoch boundary.

## 7. Resource bounds and hostile input

Each implementation MUST configure and expose to a conformance harness finite per-link and global limits for observation count, address bytes, native-route bytes, source-claim token bytes, event state, validation work, allocation retry work, table work, expiry work, and active and passive lifetimes (LINK-ADR-051). Active mode additionally bounds local claims, advertisement scheduling work, pending advertisement transmission state, and `maximum_renewal_opportunity_interval`.

After validation and before retaining a new observation, the service enforces every applicable bound (LINK-ADR-052). At capacity it MAY discard the new observation or evict an existing internal observation under a documented bounded policy. The policy may be deterministic, adaptive, or randomised; observable randomness must be controllable by a conformance harness. Eviction follows the aggregate event contract: crossing between ambiguous and usable emits `Neighbour-Changed`, while removal of the final internal observation emits `Neighbour-Expired` with `resource-evicted`.

Discovery exhaustion, collision, replay, malformed input, or eviction on one link MUST NOT allocate unbounded state, refresh unrelated entries, alter another link, or make the affected link unavailable by itself (LINK-ADR-053). Version 1 requires no discovery tombstone or sequence number. A replay may recreate an expired unauthenticated observation, but remains subject to the same validation and bounds.

## 8. Destination admission and invalidation

A `Send` destination is valid only when its current mode and kind supply exactly one usable route (LINK-ADR-054): an implicit destination on the current implicit binding; an unambiguous current unicast address-incarnation observation; broadcast with `broadcast_support` and a valid broadcast route; or multicast with `multicast_support` and a current externally configured or adapter-supplied route. Multicast support does not define group creation, identity, negotiation, cryptography, or dynamic membership.

A well-formed unicast or multicast destination whose observation, incarnation, membership, or native route is absent, expired, ambiguous, replaced, or otherwise no longer current is rejected as `rejected-stale-destination` (LINK-ADR-055). A structurally invalid kind, value, field combination, or destination unsupported by the descriptor remains `rejected-malformed`. Section 01 defines the complete `Send` precedence.

Expiry, eviction, replacement, adapter-originated route invalidation, or multicast removal establishes a destination-scoped ownership cut-off through `Shim-Route-Cutoff` or `Shim-Route-Invalidated` (LINK-ADR-056). Local claim rotation instead establishes a source-claim-scoped cut-off through `Shim-Claim-Cutoff`. Frames committed before the applicable cut-off MAY complete under the adapter profile. The bounded named `returned_frames` collection transfers every uncommitted occurrence with its exact frame, native route, and source-claim generation exactly once; service-owned affected work is cancelled, and an accepted packet with any remaining uncommitted range is classified `send-destination-invalidated` for destination invalidation or the section 06 source-claim-rotation classification for claim rotation. A failed cut-off keeps its exact route or claim generation closed without affecting unrelated work. No affected work may resume under a replacement incarnation, route, or claim generation.

The link-wide `max_packet_size` is derived from the stable worst-case addressing capability envelope, so ordinary observation arrival, expiry, ambiguity, eviction, and route replacement MUST NOT alter it (LINK-ADR-057). A change to the validated neighbour mode, adapter profile, supported address forms or lengths, MTU, or configured transmit limit that alters the envelope uses the coherent size-change barrier and one `Link-Changed` tuple. A selected destination still undergoes exact representability checks at `Send` admission, and unrelated destinations continue across selective cut-offs.

## 9. Receive processing and lifecycle

For every received generic frame, section 03 structural validation occurs first, then section 05 addressing validation and local-destination acceptance, then section 04 fragmentation and reassembly validation for Data frames (LINK-ADR-058). Implementations may fuse checks only where the primary classification, state transition, resource effect, and unrelated-state behaviour remain identical.

A received explicit unicast frame is locally acceptable only when its destination address and incarnation match a current local claim; broadcast is acceptable to an eligible receiver; multicast is acceptable only under a current local membership; and an address-less frame is acceptable only on the matching implicit binding (LINK-ADR-059). A valid frame not addressed locally is discarded as `destination-not-local` without neighbour or reassembly allocation. An incarnation mismatch is `destination-incarnation-mismatch`.

At `Link-Down`, every explicit observation, local claim, route association, expiry, ambiguity state, pending advertisement, queued neighbour event, in-flight snapshot, coalescing state, and baseline state belonging to the ending epoch is invalidated before a later epoch may use the binding (LINK-ADR-060). Detailed old-epoch events and snapshots either complete in order before the final epoch-ended state and `Link-Down`, or are superseded by `Neighbour-Baseline-Invalidated` before `Link-Down`. No old-epoch neighbour event may appear after a new epoch's authoritative baseline. Surfaced aggregates expire with `epoch-ended`. Transition to `Retired` releases every remaining section 05 resource. Nothing from the old epoch can validate a destination, source, route, or fragment in the new epoch.

## 10. Failure classifications and observability

Section 05 defines `addressing-malformed`, `destination-not-local`, `destination-incarnation-mismatch`, `neighbour-resource-exhausted`, `send-destination-invalidated`, and `source-claim-rotated` as standard asynchronous classifications, plus synchronous `Send.rejected-stale-destination` and the neighbour events and baseline-invalidated outcome above (LINK-ADR-061). Section 06 defines their bounded observable representation. Repeated malformed, non-local, mismatch, replay, or exhaustion events may be aggregated, but aggregation does not change state transitions or conceal that enumeration is required.

## 11. Conformance cases

A conformance suite MUST exercise at least the following and verify wire bytes, primary classification, observation state, destination state, ownership, event ordering, and unrelated-state isolation where applicable (LINK-ADR-062):

- every address kind at minimum and maximum length and one byte beyond;
- every permitted and forbidden TLV combination on Data and Control frames;
- implicit, unicast, broadcast, and multicast send and receive forms;
- source-less unfragmented and fragmented Data;
- equal address bytes across different links, epochs, and incarnations;
- exact observation refresh and equal address-incarnation on different routes;
- aggregate creation, usable-to-ambiguous and ambiguous-to-usable transition, partial route expiry, and final aggregate expiry;
- active advertisement at validity 1, the local maximum, and above it;
- renewal-opportunity bounds below, equal to, and above transmitted validity, including receiver clamping;
- no solicitation or withdrawal interpretation;
- passive, external, active, and implicit neighbour sources;
- observation expiry immediately before, exactly at, and after its deadline;
- enumeration ordered around observation, ambiguity, expiry, and coalescing;
- event-capacity exhaustion, sticky baseline invalidation, authoritative resynchronisation, and clearing at its observation point;
- entry, byte, route-token, event, work, and lifetime exhaustion;
- deterministic and controlled-random eviction outcomes;
- address and incarnation rotation, non-reuse, allocator exhaustion, and restart;
- pre-issue and post-issue claim collision with bounded retry, rotation, and safe failure;
- destination expiry with service-owned, shim-owned uncommitted, and committed frames;
- adapter-originated route invalidation and exact returned-work ownership;
- claim-generation cut-off across multiple routes without disturbing other generations or source-less frames;
- neighbour churn that does not alter `max_packet_size` and capability-envelope change that does;
- `Link-Down` and later `Link-Up` with old addresses, routes, events, and frames;
- every compound `Enumerate-Neighbours` rejection under the defined precedence;
- maximum 133-byte fragmented Data header and 69-byte advertisement header at MTU 256;
- spoofed, replayed, conflicting, malformed, and flooded advertisements; and
- unsupported broadcast or multicast and stale explicit unicast admission.

## 12. Security and implementation freedom

Addressing and discovery are unauthenticated and leak local metadata by design (LINK-ADR-063). An observer may correlate a source address and incarnation for their lifetime, observe destinations, replay or forge advertisements, create ambiguous claims, and exhaust bounded state. Rotation can reduce protocol-level linkability but cannot hide physical-layer identifiers, radio characteristics, timing, size, or topology. No section 05 result may bypass higher-layer authentication or authorisation.

Implementations may choose address and incarnation allocators, tables, timers, storage, schedulers, eviction policies, route-token representations, event mechanisms, and rotation policies (LINK-ADR-064). They conform only when wire bytes, field combinations, non-reuse, destination admission, cut-offs, observation semantics, bounds, ordering, lifecycle fencing, classifications, and public privacy boundaries match this section.

## 13. Requirement summary

This summary is a navigation aid. The normative prose above is authoritative.

- **LINK-ADR-001**: Section 05 defines local addressing, incarnation-qualified destinations, native-route adaptation, neighbour observation, and optional active discovery.
- **LINK-ADR-002**: Link addresses, incarnations, observations, and routes never assert or embed upper-layer identity, authentication, or reachability.
- **LINK-ADR-003**: A Link address is a typed opaque value scoped by link and epoch, with exact kind, length, and byte equality.
- **LINK-ADR-004**: Logical address kinds are unicast, broadcast, multicast, and implicit with the defined semantics.
- **LINK-ADR-005**: Wire address values use the defined kind octets and bounded opaque lengths; implicit is represented by omission.
- **LINK-ADR-006**: TLVs `0x06` through `0x0A` have the defined unique section 05 meanings and lengths.
- **LINK-ADR-007**: Destination Link Address encodes exactly one valid unicast, broadcast, or multicast form.
- **LINK-ADR-008**: Source Link Address is unicast with 1 through 32 opaque bytes.
- **LINK-ADR-009**: Destination and sender incarnations are exact non-zero opaque 16-byte values.
- **LINK-ADR-010**: Advertisement Validity is a non-zero unsigned 32-bit big-endian duration in seconds.
- **LINK-ADR-011**: Explicit unicast requires destination address and incarnation; broadcast and multicast forbid destination incarnation.
- **LINK-ADR-012**: Implicit-peer Data omits destination fields; omission elsewhere is forbidden unless a Control definition permits it.
- **LINK-ADR-013**: Every transmitted Data frame uses exactly one valid destination form and required capability.
- **LINK-ADR-014**: Data source is optional; a present source requires sender incarnation.
- **LINK-ADR-015**: Fragmented Data carries sender incarnation unless a safe implicit-peer incarnation is supplied by the binding.
- **LINK-ADR-016**: Each Control Type defines its complete permitted addressing form.
- **LINK-ADR-017**: Malformed addressing is discarded before neighbour, route, fragmentation, or reassembly allocation.
- **LINK-ADR-018**: The maximum mandatory offset-zero fragmented Data header is 133 bytes, leaving 123 bytes at MTU 256.
- **LINK-ADR-019**: The descriptor contains exactly one required neighbour mode from the four defined values.
- **LINK-ADR-020**: `none` supplies no proactive unicast destination and has the defined applicability.
- **LINK-ADR-021**: `implicit-peer` uses one binding-inherent peer and a restart-safe peer-incarnation rule.
- **LINK-ADR-022**: `externally-supplied` uses bounded injectable address-incarnation-route entries.
- **LINK-ADR-023**: `active-advertisement` requires send and receive, broadcast support, and adapter route support.
- **LINK-ADR-024**: One-way direction restricts neighbour mode as specified.
- **LINK-ADR-025**: A local address-incarnation pair is not issued twice on one link and epoch or reissued during the host run.
- **LINK-ADR-026**: Incarnation allocation changes or fails safely before wrap, repetition, loss, or unpreserved restart.
- **LINK-ADR-027**: Allocation remains method-neutral but checks current claims and observations, bounds retry, and safely handles collisions before and after issue.
- **LINK-ADR-028**: Active discovery supports controlled address-and-incarnation rotation without a universal interval.
- **LINK-ADR-029**: Rotation creates no continuity claim and invalidates the old claim through the defined cut-off.
- **LINK-ADR-030**: A native route is an opaque bounded epoch-scoped adapter token, not identity or a Network-visible value.
- **LINK-ADR-031**: Shim receive supplies an observed native route where available without parsing Link semantics.
- **LINK-ADR-032**: Shim send consumes the selected native route and source-claim generation metadata without interpreting either as wire identity.
- **LINK-ADR-033**: Adapter profiles define finite route and source-claim token representation, mapping, lifetime, commitment, cut-off, return, and invalidation.
- **LINK-ADR-034**: Size-change return preserves each uncommitted frame and route together; down cancels both.
- **LINK-ADR-035**: Shims act on native routes and remain unnecessary as Link-TLV or payload parsers.
- **LINK-ADR-036**: Control Type `0x0001` is the exact empty-body Neighbour Advertisement form.
- **LINK-ADR-037**: Advertisements contain only ephemeral local addressing and validity, never identity or continuity data.
- **LINK-ADR-038**: Active profiles define a finite renewal-opportunity bound strictly below transmitted validity and guarantee an opportunity, not reception.
- **LINK-ADR-039**: Effective advertisement lifetime is clamped to a finite local maximum that may intentionally permit observation gaps.
- **LINK-ADR-040**: Version 1 has no solicitation, discovery response, or withdrawal.
- **LINK-ADR-041**: Explicit observations are keyed by link, epoch, address, incarnation, and native route; exact refresh affects only that entry.
- **LINK-ADR-042**: Equal address-incarnation on different routes is ambiguous and unusable until only one current route remains.
- **LINK-ADR-043**: Semantically valid source-bearing Data may create a finite passive observation before reassembly retention but never from malformed fragments.
- **LINK-ADR-044**: External observations use the common bounded, ambiguous, event, and cut-off semantics without becoming identity.
- **LINK-ADR-045**: An implicit peer is binding state fenced by binding and epoch rather than an explicit table entry.
- **LINK-ADR-046**: `Neighbour-Observed` exposes one state-bearing public aggregate per address-incarnation without native routes or trust assertions.
- **LINK-ADR-047**: `Neighbour-Changed` reports the complete updated aggregate record and `Neighbour-Expired` reports only final aggregate disappearance with one reason.
- **LINK-ADR-048**: `Enumerate-Neighbours` applies defined rejection precedence and returns an authoritative bounded aggregate snapshot without side effects.
- **LINK-ADR-049**: Events and snapshots remain ordered, with bounded baseline invalidation and enumeration recovery when exact history cannot be retained.
- **LINK-ADR-050**: Expiry wins at or after its deadline before refresh, query, or destination use.
- **LINK-ADR-051**: Observation, address, route, claim-token, event, retry, scheduling, work, and lifetime resources have finite exposed bounds.
- **LINK-ADR-052**: Admission enforces bounds and may discard or use a bounded test-controllable eviction policy whose events follow aggregate transitions.
- **LINK-ADR-053**: Hostile discovery input remains bounded and isolated; no tombstone or sequence number creates a false replay guarantee.
- **LINK-ADR-054**: Send destinations are valid only through one current usable implicit, unicast, broadcast, or multicast route.
- **LINK-ADR-055**: Well-formed stale destinations return `rejected-stale-destination`; malformed or unsupported forms remain malformed.
- **LINK-ADR-056**: Route and claim invalidation use their exact selective cut-offs, permit committed completion, and transfer or abandon affected work once.
- **LINK-ADR-057**: Stable worst-case capacity ignores neighbour churn; only capability-envelope changes use the coherent size-change barrier.
- **LINK-ADR-058**: Receive processing orders section 03 structure, section 05 addressing and destination, then section 04 fragmentation.
- **LINK-ADR-059**: Local destination acceptance and non-local or incarnation-mismatch classifications are exact and pre-allocation.
- **LINK-ADR-060**: Link down orders or invalidates neighbour histories and fences and releases every section 05 state item across epochs.
- **LINK-ADR-061**: Section 05 defines its asynchronous classifications, synchronous rejection, neighbour events, and baseline invalidation for section 06 representation.
- **LINK-ADR-062**: Conformance exercises the required wire, mode, observation, expiry, resource, collision, lifecycle, cut-off, and hostile-input cases.
- **LINK-ADR-063**: Addressing and discovery are unauthenticated and metadata-visible and never replace higher-layer security.
- **LINK-ADR-064**: Internal allocation, storage, timing, routing-token, event, and policy mechanisms remain implementation-defined behind observable invariants.
