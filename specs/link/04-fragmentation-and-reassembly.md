# 04. Fragmentation and Reassembly

## 1. Purpose

This section defines how the Link layer carries one opaque Network packet in one or more canonical Data frames, how it reports maximum packet size, and how a receiver reconstructs fragmented packets without ambiguous overlap or unbounded state.

Fragmentation is internal to the Link service and invisible to the Network layer (LINK-FRG-001). One accepted `Send` carries one complete Network payload, and one successful `Receive` carries one complete reassembled payload. No partial payload crosses the Network service boundary.

Shim-native subdivision is not Link fragmentation (LINK-FRG-002). A shim may map one complete canonical frame to several native units under section 02, but it reconstructs the byte-identical frame before `Shim-Receive`. This section operates only on complete canonical frames.

Requirement IDs in this section use the `LINK-FRG-NNN` prefix.

## 2. Data-frame forms

### 2.1 Unfragmented form

An unfragmented Data frame contains none of the fragmentation TLVs allocated by this section, and its body is the complete Network packet (LINK-FRG-003). A non-empty Network packet that fits the selected unfragmented frame shape under the current MTU MUST use this form.

### 2.2 Fragmented form

Every fragmented Data frame contains Packet Identifier, Packet Length, and Fragment Offset TLVs. The fragment whose offset is zero additionally contains Packet CRC-32C; every non-zero-offset fragment forbids Packet CRC-32C (LINK-FRG-004). Packet Identifier alone is also permitted on an unfragmented Data frame when section 06 reliable delivery applies; its complete body is the packet and Packet Length, Fragment Offset, and Packet CRC-32C are forbidden. Every other partial combination is malformed.

Version 1 allocates these TLV types (LINK-FRG-005):

| Type | Name | Value length | Encoding |
|---:|---|---:|---|
| `0x02` | Packet Identifier | 8 | unsigned big-endian |
| `0x03` | Packet Length | 2 | unsigned big-endian |
| `0x04` | Fragment Offset | 2 | unsigned big-endian |
| `0x05` | Packet CRC-32C | 4 | unsigned big-endian |

These assignments occupy the section 03 version 1 TLV namespace. Later sections MUST NOT allocate another meaning to types `0x02` through `0x05`.

### 2.3 Packet Identifier

Packet Identifier is scoped to one sender incarnation and identifies one complete Network packet (LINK-FRG-006). It identifies all fragments when fragmentation is used and, under section 06, identifies an unfragmented reliable packet and its acknowledgement. It is not a global identifier and requires no central allocator.

A sender MUST NOT reuse a Packet Identifier during one sender incarnation (LINK-FRG-007). The prohibition applies across fragmented and unfragmented reliable packets. It MAY allocate identifiers using a monotonic counter, collision-checked random selection, or another method satisfying that rule. Before allocation state could wrap, repeat, or be lost, the sender MUST establish a new incarnation discriminator. This prohibition is deliberately stronger than attempting to estimate every receiver's retention lifetime.

### 2.4 Packet Length and Fragment Offset

Packet Length is the length in bytes of the complete original Network packet and MUST be from 1 through 65,535 inclusive (LINK-FRG-008).

Fragment Offset is the zero-based byte position of the first fragment-body byte within the original packet (LINK-FRG-009). A fragment body MUST be non-empty, its offset MUST be less than Packet Length, and `offset + fragment_body_length` MUST be no greater than Packet Length. The addition MUST be checked without integer wrap.

A fragmented frame whose body alone covers the complete declared packet is malformed. A valid fragmented packet therefore contains at least two accepted non-duplicate fragments.

### 2.5 Packet CRC-32C

The offset-zero fragment MUST contain Packet CRC-32C exactly once, and a non-zero-offset fragment MUST NOT contain it (LINK-FRG-010). The value is the CRC-32C of the complete original Network packet before fragmentation.

Packet CRC-32C uses the Castagnoli parameters defined in section 03 and covers exactly the Packet Length opaque payload bytes in increasing byte-offset order (LINK-FRG-011). It detects accidental identifier collision or incorrect reassembly but provides no authentication, confidentiality, replay protection, or defence against a deliberate forger.

### 2.6 Body and padding

The fragment body is the Data-frame body after section 03 has removed explicit Link padding (LINK-FRG-012). Padding bytes never occupy packet offsets and never contribute to Packet Length or Packet CRC-32C.

## 3. Maximum packet size

### 3.1 Global and guaranteed bounds

Every send-capable Link implementation MUST be capable of accepting a Network packet of at least 256 bytes for transmission, and every receive-capable Link implementation MUST be capable of reassembling a Network packet of at least 256 bytes, when the applicable bounded resources are available (LINK-FRG-013). These are direction-qualified guarantees, not required allocations for every concurrent transmission or context. A receive-only link is exempt from the transmit guarantee, and a send-only link is exempt from the reassembly guarantee.

The global maximum Network packet size is 65,535 bytes (LINK-FRG-014). No sender may accept, fragment, or declare a larger packet.

### 3.2 Meaning of the reported value

The `max_packet_size` reported by a link is the largest Network payload the local Link service can accept, fragment, and submit for every destination and source form within the current validated capability envelope under its MTU, mandatory headers, fragment-count ceiling, and bounded local transmit configuration (LINK-FRG-015). It is a stable conservative local send capability, not a promise of remote receipt, remote reassembly capacity, or delivery. A receive-only link MUST report `max_packet_size` as zero. A send-capable link with no usable destination MAY still report the value derived from its validated capability envelope, although individual `Send` calls remain subject to section 05 destination admission. Zero is a no-send sentinel and is not a valid Network payload size.

Every receive-capable implementation MUST configure a local fragmented-packet reassembly limit from 256 through 65,535 inclusive (LINK-FRG-016). A fragmented packet declaring a larger Packet Length is discarded as `reassembly-packet-size-exceeded`. The receive limit is a local resource bound and is not added to the section 02 capability descriptor by this section.

### 3.3 Capacity derivation

For a send-capable link, the Link service determines the body capacity under section 03 for every destination form, source form, maximum permitted address length, and required fragmented-frame shape that the current validated descriptor, neighbour mode, and adapter profile can produce. Let `minimum_fragment_body_capacity` be the smallest positive capacity across that stable capability envelope, including the larger offset-zero header carrying Packet CRC-32C (LINK-FRG-017). Optional Link padding MUST be omitted or reduced when necessary and MUST NOT reduce the reported capability. Observation arrival, refresh, expiry, ambiguity, eviction, or route replacement does not change this envelope.

The reported value MUST satisfy (LINK-FRG-018):

```text
max_packet_size = min(
    configured_transmit_packet_limit,
    65,535,
    256 * minimum_fragment_body_capacity
)
```

The multiplication MUST be evaluated without integer wrap. For a send-capable link, the configured transmit packet limit and calculated result MUST each be at least 256. A smaller configured limit is invalid for an available send-capable tuple and MUST NOT be silently clamped. Section 05 defines addressing overhead and MUST preserve at least one fragment-body byte for every form in the validated capability envelope at the guaranteed 256-byte MTU (LINK-FRG-019). A send-capable link that cannot satisfy the guaranteed packet size for that envelope cannot become or remain `Available` with that tuple. The formula does not apply when `max_packet_size` is the zero sentinel.

A change to MTU, the validated capability envelope, mandatory header overhead, or configured transmit limit that changes `max_packet_size` is surfaced through the coherent `Link-Changed` tuple and size-change barrier defined in sections 01 and 02 (LINK-FRG-020). Ordinary neighbour-table churn MUST NOT change the reported value.

## 4. Send fragmentation

### 4.1 Atomic admission

Before returning `accepted` for a `Send`, the Link service MUST atomically establish all of the following (LINK-FRG-021):

- the packet is non-empty and no larger than current `max_packet_size`;
- the packet is representable by at most 256 fragments under the current tuple and selected destination;
- a non-reused Packet Identifier is available when fragmentation is required;
- the packet bytes can remain immutable while owned for transmission;
- bounded state is available to retain or reference the packet and track every unsubmitted range;
- the supplied link, epoch, and destination pass the section 01 admission rules.

If any required local transmit resource is unavailable, `Send` MUST return `rejected-queue-full` before any fragment is emitted (LINK-FRG-022). Other overlapping rejection conditions retain the precedence defined by LINK-SVC-090.

After acceptance, the Packet Identifier, Packet Length, Packet CRC-32C, destination, and payload bytes MUST remain fixed for that packet's transmission ownership (LINK-FRG-023).

An implementation MAY retain one packet representation and construct frames incrementally, use non-contiguous or scatter-gather storage, pre-build frames, or use another bounded representation (LINK-FRG-024). It is not required to retain 256 complete frame copies.

### 4.2 Frame construction

If the complete packet fits the selected unfragmented frame shape, the Link service MUST emit one unfragmented Data frame (LINK-FRG-025). Otherwise it allocates a Packet Identifier, calculates Packet CRC-32C once, and emits fragmented Data frames covering the packet.

Every non-final fragment in emission order SHOULD use the maximum body capacity available to its actual frame shape (LINK-FRG-026). A sender MAY use a smaller non-empty body for a documented operational reason, or where an earlier committed range or size change requires a smaller uncovered range to preserve exact non-overlapping offsets. Before acceptance, the sender MUST establish that its actual selected partition remains representable within the current frame bounds and the 256-fragment ceiling. The final fragment carries every remaining byte. No accepted packet may require more than 256 distinct fragments (LINK-FRG-027).

Fragment emission order is implementation-defined and carries no receive semantic (LINK-FRG-028). A receiver accepts arbitrary arrival order. Section 07 owns any broader ordering guarantee.

### 4.3 Shim admission

A transient `Shim-Send.rejected-queue-full` leaves the affected range unsubmitted. The Link service MAY retry shim admission within its finite bounded scheduling rules without treating that retry as medium retransmission (LINK-FRG-029). It MUST NOT create unbounded waiting, work, or duplicate range ownership.

An intrinsic `Shim-Send.rejected-malformed` for a service-generated fragment is a local implementation or changed-state fault. The Link service MUST stop attempting the remaining packet and classify it as `send-fragmentation-abandoned` rather than retrying indefinitely (LINK-FRG-030).

### 4.4 MTU and capability changes

An MTU increase does not invalidate already constructed frames; unsubmitted work MAY remain in its existing valid form (LINK-FRG-031). New admissions use the new coherent tuple after `Link-Changed`.

For an MTU or maximum-packet-size decrease, the size-change barrier classifies work against one side of its cut-off (LINK-FRG-032). Committed fragments complete under the old tuple. For best-effort packets, uncommitted frames are returned to Link framing control and their uncovered byte ranges are reframed under the new tuple using the same Packet Identifier, Packet Length, Packet CRC-32C, destination, and offsets. Bytes already committed MUST NOT be emitted again solely because the tuple changed. For a section 06 reliable fixed frame plan, returned occurrences are not reframed: the plan continues byte-identically only when it and all governing reliability semantics remain valid under the new tuple, otherwise the transaction terminalises and the returned occurrences are cancelled.

If the remaining best-effort uncovered ranges cannot complete within the new frame-size, packet-size, or 256-fragment bounds, the service MUST abandon them as `send-invalidated-by-size-change` (LINK-FRG-033). Acceptance was best-effort and is not converted into a synchronous result, but the abandonment is observable under section 06. A reliable transaction invalidated by the change instead follows the section 06 terminal result and `size-change-invalidated` reason rules.

## 5. Receive validation and reassembly

### 5.1 Reassembly key and sender incarnation

The reassembly key is the tuple `(link_id, availability_epoch, sender_incarnation_discriminator, Packet Identifier)` (LINK-FRG-034). No context or tombstone is shared across a different tuple component.

For a multi-sender link, the section 05 Sender Incarnation is the reassembly discriminator and MUST change before a sender can lose and reuse its Packet Identifier allocation state (LINK-FRG-035). It may be packet-scoped when no source address is present. An `implicit-peer` profile may instead supply its binding-specific peer incarnation under LINK-ADR-021. A source address without its incarnation is insufficient.

### 5.2 Validation before allocation

The Link service MUST complete section 03 framing and section 05 addressing, incarnation, expiry, and local-destination validation before applying the following section 04 semantic order or allocating or extending a reassembly context (LINK-FRG-036):

1. validate the complete fragmentation-field form, field lengths, field placement, and non-empty body;
2. expire any context or tombstone whose deadline is reached at the frame's logical observation point;
3. test the complete reassembly key against retained tombstones;
4. validate decoded field values, Packet Length, and non-wrapping range bounds;
5. enforce the local fragmented-packet receive limit;
6. compare fixed context metadata;
7. test for an exact duplicate;
8. test for overlap or conflict;
9. enforce the 256-fragment limit;
10. enforce resource admission; and
11. retain the fragment and test for completion.

An implementation MAY fuse or short-circuit checks only when it preserves the same primary classification, state transition, and unrelated-state behaviour. A completion-tombstone match silently discards the fragment as an already delivered duplicate, except that a live section 06 reliable terminal record diverts a structurally and semantically valid matching retry to the bounded acknowledgement-repetition rule after framing and addressing validation and before ordinary completion-tombstone discard. A conflict or integrity tombstone match discards the fragment with the tombstone's stored terminal classification. No tombstone or reliable terminal-record match allocates or extends a reassembly context, extends either deadline, or retains fragment bytes.

The first accepted non-tombstoned fragment creates a context and fixes its key, Packet Length, creation time, expiry, and any Packet CRC-32C present on offset zero (LINK-FRG-037). A non-zero-offset fragment may create the context before the expected CRC is known. The offset-zero fragment fixes the CRC when it arrives.

### 5.3 Arrival order and duplicates

Fragments MAY arrive in any order (LINK-FRG-038). Completion depends on byte-range coverage, not arrival or emission order.

An exact duplicate has the same reassembly key, Packet Length, Fragment Offset, body length, body bytes, and, where present, Packet CRC-32C (LINK-FRG-039). It is ignored without allocating another body copy or fragment slot, changing coverage, counting as progress, extending expiry, or causing another delivery.

Any of the following is a reassembly conflict (LINK-FRG-040):

- a different Packet Length for the same key;
- an offset-zero fragment carrying a different Packet CRC-32C;
- identical offset and length with different body bytes;
- any partial overlap;
- containment that is not an exact duplicate;
- inconsistent fixed metadata.

Even byte-identical partial overlap is forbidden. A receiver MUST NOT use first-wins, last-wins, or implementation-defined overlap resolution.

On conflict, the receiver MUST discard the complete context, deliver nothing, install a conflict tombstone carrying `reassembly-conflict` through the context's original expiry, and classify `reassembly-conflict` (LINK-FRG-041).

### 5.4 Completion

A context is complete only when at least two accepted non-duplicate fragments cover every byte in `[0, Packet Length)` exactly once, with no gap, and the offset-zero Packet CRC-32C is known (LINK-FRG-042).

Before `Receive`, the Link service MUST calculate CRC-32C over the reconstructed bytes in increasing offset order and compare it with Packet CRC-32C (LINK-FRG-043). Mismatch discards the context, installs a conflict tombstone carrying `reassembly-integrity-failed` through the original expiry, and classifies `reassembly-integrity-failed`.

On a match, the receiver MUST install a completion tombstone before making the complete opaque packet observable through one `Receive` invocation (LINK-FRG-044). For a fragmented reliable transaction, the authoritative section 06 terminal record satisfies this completion-tombstone obligation and follows section 06's identity, lifetime, acknowledgement, capacity, and non-eviction rules. Otherwise, the completion tombstone retains the complete reassembly key through the expiry that the completed context would have had, retains no fragment body, and uses the same non-refreshing deadline. The receiver then releases fragment storage as soon as permitted by the service representation.

One fragmented-packet identity produces at most one `Receive` while its completion tombstone or composed section 06 reliable terminal record remains present (LINK-FRG-045). This does not claim general duplicate suppression: the same payload under a new Packet Identifier and unfragmented duplicate frames remain governed by section 07.

## 6. Resource bounds and lifecycle

### 6.1 Required local bounds

Each implementation MUST configure and expose to a conformance harness finite limits for, at minimum (LINK-FRG-046):

- fragmented Packet Length accepted on receive, from 256 through 65,535;
- concurrent contexts per link;
- aggregate retained packet bytes per link;
- fragment-range metadata per context and per link;
- conflict and completion tombstones per link, with capacity for at least one terminal tombstone or equivalent bounded suppression state on every available receive-capable link;
- validation, comparison, CRC, eviction, and expiry work;
- reassembly lifetime or progress as defined below.

No packet may contain more than 256 accepted non-duplicate fragments. All storage and work MUST remain isolated per link and globally bounded by the implementation.

### 6.2 Lifetime

Where a binding has a defensible maximum inter-fragment interval, its absolute reassembly lifetime begins when the first fragment creates the context and is (LINK-FRG-047):

```text
maximum_reassembly_lifetime =
    256 * maximum_inter_fragment_interval
    + finite_scheduling_allowance
```

The interval and allowance MUST be finite, configured, and exposed to a conformance harness. Arrival of a new fragment or exact duplicate MUST NOT refresh or extend the absolute expiry. At a logical observation time equal to or later than a context, tombstone, or progress-rule deadline, expiry MUST be processed before the observed fragment. Timer resolution and implementation mechanism remain implementation-defined, but the exposed deadline and boundary outcome MUST be reproducible.

An intermittent binding without a defensible finite interval MUST define a finite absolute retention limit or a finite progress-and-abandonment rule that bounds both time and retained state for an incomplete context (LINK-FRG-048). The rule MUST be testable and MUST NOT permit indefinite retention.

### 6.3 Resource admission and eviction

After validation and before retaining a fragment, the receiver MUST enforce packet, fragment-count, context, byte, metadata, tombstone, and work limits (LINK-FRG-049). Capacity exhaustion may discard the new fragment or evict an implementation-selected incomplete context and is classified `reassembly-resource-exhausted`.

The local eviction policy is implementation-defined and MAY be deterministic, adaptive, or randomised, but it MUST be documented, bounded, isolated to the affected link, and incapable of evicting a completed packet already being delivered merely to admit incomplete work (LINK-FRG-050). Any randomness affecting conformance-observable outcomes MUST be injectable, seedable, or otherwise controllable by a conformance harness. An evicted live key SHOULD receive a bounded conflict-style tombstone when capacity permits, preventing immediate reallocation churn. Exact duplicate input consumes no new fragment or body capacity.

Conflict, integrity, and completion tombstones are mandatory terminal states. Each retains the complete reassembly key, its fixed original-context expiry, and, for a conflict or integrity tombstone, its terminal classification. A composed section 06 reliable terminal record is a completion tombstone for these requirements but remains governed by section 06's longer lifetime and non-eviction rule. When ordinary tombstone capacity is full, installation MUST atomically replace an existing evictable tombstone selected by the documented bounded eviction policy. Replacement MUST NOT evict a live reliable terminal record or affect a completed packet already being delivered. Eviction may permit later reallocation or duplicate delivery after the evicted suppression state is lost; section 07 owns any stronger advertised duplicate-suppression guarantee.

### 6.4 Availability epochs and retirement

At the receive cut-off for `Link-Down`, the Link service MUST discard every reassembly context and tombstone belonging to the ending availability epoch and classify incomplete contexts as `reassembly-epoch-ended` (LINK-FRG-051). At the same cut-off, every accepted fragmented transmission with any uncommitted range MUST be terminally abandoned as `send-fragmentation-abandoned`; its uncommitted packet ownership and tracking state MUST be released within a finite bound and MUST NOT cross into a later availability epoch. Frames committed before the cut-off MAY complete under the applicable adapter profile, as permitted by section 01, but no uncommitted range may resume in a later epoch. No fragment, byte, CRC, range, tombstone, or unfinished transmit ownership from the ending epoch may contribute to a later epoch.

Transition to `Retired` MUST release every remaining reassembly context, tombstone, packet buffer, range record, timer, and related resource for the binding (LINK-FRG-052), restating LINK-SVC-085.

A receive-direction MTU reduction does not invalidate fragment bytes admitted before the new bound (LINK-FRG-053). New frames obey the new operative receive bound. A context that cannot complete under later input remains subject to its unchanged expiry and resource rules.

## 7. Failure classifications and observability

Section 04 defines these classifications (LINK-FRG-054):

| Classification | Meaning |
|---|---|
| `fragment-malformed` | Fragmentation fields or body violate section 2 or section 05 sender-discriminator requirements. |
| `reassembly-packet-size-exceeded` | Declared Packet Length exceeds the local fragmented-packet receive limit. |
| `reassembly-fragment-limit-exceeded` | Another distinct fragment would exceed 256 accepted fragments. |
| `reassembly-conflict` | Metadata, CRC declaration, duplicate bytes, or ranges conflict. |
| `reassembly-integrity-failed` | Complete reconstructed bytes fail Packet CRC-32C. |
| `reassembly-resource-exhausted` | Bounded receive state cannot admit work or an incomplete context is evicted. |
| `reassembly-expired` | An incomplete context reaches its absolute lifetime or progress limit. |
| `reassembly-epoch-ended` | An incomplete context is discarded at the ending availability epoch. |
| `send-invalidated-by-size-change` | A reduced coherent tuple makes accepted remaining ranges unrepresentable. |
| `send-fragmentation-abandoned` | Another post-acceptance local condition prevents further attempts. |

Each classification is isolated to its packet or affected context and MUST NOT by itself change link availability or affect unrelated packets, flows, contexts, or links (LINK-FRG-055).

Every classification MUST be observable through the standard mechanism defined in section 06 (LINK-FRG-056). Section 06 may represent classifications through counters, indications, accumulated metadata, or a combination. Repeated tombstone matches MAY be aggregated rather than producing an unbounded event for every fragment. Exact duplicates ignored within a live context are not failures.

## 8. Language-neutral conformance vectors

Hexadecimal octets are separated by spaces. These vectors are normative examples (LINK-FRG-057).

### 8.1 Two-fragment packet

This vector uses MTU 256 and a valid point-to-point frame shape requiring no additional Data TLVs. The original opaque Network packet is the 256 octets whose values increase from `00` through `FF`. Its Packet Length is 256, its Packet CRC-32C is `9C 44 18 4B`, and Packet Identifier is `0x0000000000000001`.

The offset-zero frame is exactly 256 bytes:

```text
fixed header:       01 01 1B
Packet Identifier:  02 08 00 00 00 00 00 00 00 01
Packet Length:      03 02 01 00
Fragment Offset:    04 02 00 00
Packet CRC-32C:     05 04 9C 44 18 4B
fragment body:      every increasing octet from 00 through E4 inclusive
```

The non-zero frame is exactly 48 bytes:

```text
fixed header:       01 01 15
Packet Identifier:  02 08 00 00 00 00 00 00 00 01
Packet Length:      03 02 01 00
Fragment Offset:    04 02 00 E5
fragment body:      every increasing octet from E5 through FF inclusive
```

Either arrival order reconstructs exactly the increasing 256-octet sequence. Repeating either frame byte-identically before completion is an ignored exact duplicate.

### 8.2 Invalid and conflicting cases

Each case is independent:

| Condition | Expected classification |
|---|---|
| only Packet Identifier and Packet Length present | `fragment-malformed` |
| zero-byte fragment body | `fragment-malformed` |
| Packet Length zero | `fragment-malformed` |
| offset equal to Packet Length | `fragment-malformed` |
| offset plus body length exceeds Packet Length | `fragment-malformed` |
| offset zero without Packet CRC-32C | `fragment-malformed` |
| non-zero offset with Packet CRC-32C | `fragment-malformed` |
| one fragment covers the complete declared packet | `fragment-malformed` |
| same key with different Packet Length | `reassembly-conflict` |
| same offset and length with different bytes | `reassembly-conflict` |
| byte-identical partial overlap | `reassembly-conflict` |
| 257th distinct otherwise-valid fragment | `reassembly-fragment-limit-exceeded` |
| declared length one byte above local receive limit | `reassembly-packet-size-exceeded` |
| complete coverage with CRC mismatch | `reassembly-integrity-failed` |

### 8.3 Required state and boundary cases

A conformance suite MUST additionally exercise at least (LINK-FRG-058):

- packets of 1, 255, 256, the reported maximum, 65,535, and one byte above each applicable maximum;
- exact unfragmented capacity and one byte above it;
- minimum MTU with the largest mandatory Data header permitted by section 05;
- first, middle, and final fragments of unequal lengths;
- every permutation of at least three fragments;
- a gap immediately before completion;
- exact duplicate, conflicting duplicate, partial overlap, and containment;
- identifier equality under different senders, links, and availability epochs;
- context, byte, metadata, tombstone, and work exhaustion;
- arrival immediately before, exactly at, and after expiry, including compound expiry and otherwise-invalid input;
- completion tombstone replay before and after expiry;
- conflict and integrity tombstone matches, tombstone-capacity replacement, and replay after replacement;
- link down and later up with old fragments presented after the cut-off;
- link down with a fragmented transmission containing committed and uncommitted ranges;
- MTU increase and decrease with service-owned, shim-owned uncommitted, and committed fragments;
- compound-invalid frames covering every adjacent pair of the section 5.2 validation order;
- send-only, receive-only, and send-capable links with an empty valid destination set;
- broadcast and multicast destinations with different addressing overhead;
- CRC calculation independent of fragment sizes and arrival order.

Each invalid case MUST identify its primary validation failure, resulting context or tombstone state, observable classification, and whether unrelated state remains unchanged.

## 9. Security and implementation freedom

Fragmentation is exposed to unauthenticated hostile input. An attacker can create incomplete contexts, spoof identifiers, inject conflicts, replay fragments, force CRC work, and exhaust bounded state. CRC-32C detects accidental error only. It cannot establish sender identity or prevent a deliberate conflict or denial of service.

Implementations may choose buffers, range sets, bitmaps, interval trees, sparse storage, external storage, schedulers, eviction policies, timer structures, and incremental CRC strategies. They conform only when wire bytes, coverage, duplicate and conflict outcomes, bounds, lifetime, lifecycle fencing, classifications, and observable service behaviour match this section.

## 10. Requirement summary

This summary is a navigation aid. The normative prose above is authoritative.

- **LINK-FRG-001**: Fragmentation is internal to Link; one `Send` and one successful `Receive` each carry one complete opaque Network payload.
- **LINK-FRG-002**: Shim-native subdivision is distinct and reconstructs complete byte-identical canonical frames before Link processing.
- **LINK-FRG-003**: An unfragmented Data frame has no fragmentation TLVs and carries the complete packet; a fitting packet uses this form.
- **LINK-FRG-004**: Fragmented frames carry Identifier, Length, Offset, and offset-zero CRC; Identifier alone is permitted only for unfragmented reliable Data; other partial combinations are malformed.
- **LINK-FRG-005**: Version 1 allocates TLV types `0x02` through `0x05` as defined in section 2.2.
- **LINK-FRG-006**: Packet Identifier is sender-incarnation scoped, centrally unallocated, and identifies one fragmented or unfragmented reliable packet.
- **LINK-FRG-007**: A sender never reuses a Packet Identifier across fragmented or reliable uses within one incarnation and changes incarnation before allocation state can repeat or be lost.
- **LINK-FRG-008**: Packet Length is the complete packet byte length from 1 through 65,535.
- **LINK-FRG-009**: Fragment Offset and non-empty body define a non-wrapping range wholly inside Packet Length; one fragment cannot cover a complete fragmented packet.
- **LINK-FRG-010**: Packet CRC-32C occurs exactly on offset zero.
- **LINK-FRG-011**: Packet CRC-32C covers the complete original payload using section 03 parameters and is accidental-error detection only.
- **LINK-FRG-012**: Fragment body excludes Link padding, which occupies no packet range and no packet CRC input.
- **LINK-FRG-013**: Send-capable implementations accept at least a 256-byte packet for transmission and receive-capable implementations reassemble at least a 256-byte packet when bounded resources are available.
- **LINK-FRG-014**: Global maximum packet size is 65,535 bytes.
- **LINK-FRG-015**: Reported `max_packet_size` is a stable conservative local send capability across the validated addressing envelope; receive-only links report zero.
- **LINK-FRG-016**: Each receiver configures a fragmented-packet reassembly limit from 256 through 65,535; larger declarations are discarded.
- **LINK-FRG-017**: Minimum fragment-body capacity is conservative across every form in the stable validated capability envelope, including offset zero, without optional padding reduction.
- **LINK-FRG-018**: A send-capable reported maximum is the specified minimum of configured limit, global maximum, and 256 times minimum fragment capacity; an available non-zero send capability is never silently clamped below 256.
- **LINK-FRG-019** (specification-suite invariant): Section 05 preserves positive fragment capacity for the capability envelope at minimum MTU; a link unable to provide the guaranteed packet size cannot be Available.
- **LINK-FRG-020**: Only a change to the capacity envelope or its sizing inputs changes the reported value and uses the coherent size-change barrier.
- **LINK-FRG-021**: `Send` atomically validates and reserves all state required for whole-packet ownership before acceptance.
- **LINK-FRG-022**: Local transmit resource failure returns `rejected-queue-full` before fragment emission.
- **LINK-FRG-023**: Accepted packet identifier, length, CRC, destination, and bytes remain fixed during transmission ownership.
- **LINK-FRG-024**: Internal packet and frame storage representation remains implementation-defined and bounded.
- **LINK-FRG-025**: A packet fitting one selected frame uses unfragmented form; otherwise fragmentation allocates identity and CRC.
- **LINK-FRG-026**: Non-final fragments should fill actual capacity, but documented operational reasons may use a smaller valid partition proven at admission to remain within 256 fragments.
- **LINK-FRG-027**: No packet uses more than 256 distinct fragments.
- **LINK-FRG-028**: Fragment emission and arrival order carry no reassembly semantic.
- **LINK-FRG-029**: Transient shim queue exhaustion may retry bounded admission without unbounded work or duplicate range ownership.
- **LINK-FRG-030**: Intrinsic malformed rejection of a service-generated fragment observably abandons the packet without indefinite retry.
- **LINK-FRG-031**: MTU increase does not invalidate existing valid frames; new admissions use the new tuple.
- **LINK-FRG-032**: Size reduction lets committed frames complete, reframes best-effort uncovered ranges, and applies byte-identical continue-or-terminalise handling to reliable fixed plans.
- **LINK-FRG-033**: An unrepresentable best-effort remainder is observably abandoned, while reliable invalidation follows section 06 outcome semantics.
- **LINK-FRG-034**: Reassembly key is link, local epoch, sender incarnation, and Packet Identifier.
- **LINK-FRG-035** (specification-suite invariant): Section 05 Sender Incarnation or the implicit-peer profile supplies the restart-safe reassembly discriminator.
- **LINK-FRG-036**: Sections 03 and 05 validation precedes the specified section 04 semantic order and every allocation or extension.
- **LINK-FRG-037**: The first accepted fragment fixes context identity, Packet Length, creation, expiry, and CRC when present; offset zero may arrive later.
- **LINK-FRG-038**: Fragments may arrive in arbitrary order.
- **LINK-FRG-039**: Exact duplicates allocate nothing, change nothing, count as no progress, and cause no delivery.
- **LINK-FRG-040**: Inconsistent metadata, CRC, bytes, partial overlap, or non-exact containment is conflict; overlap resolution is never implementation-defined.
- **LINK-FRG-041**: Conflict discards and tombstones the context through original expiry with stored classification `reassembly-conflict`.
- **LINK-FRG-042**: Completion requires at least two accepted fragments and exact gap-free coverage with known CRC.
- **LINK-FRG-043**: CRC is verified before `Receive`; mismatch discards and tombstones through original expiry with stored classification `reassembly-integrity-failed`.
- **LINK-FRG-044**: Passing reassembly installs completion suppression before one packet becomes observable; a reliable terminal record composes this state under section 06.
- **LINK-FRG-045**: One fragmented identity under a completion tombstone or composed reliable terminal record produces at most one `Receive`; broader duplicate suppression remains section 07-owned.
- **LINK-FRG-046**: Implementations expose finite packet, context, byte, metadata, tombstone, work, and lifetime limits; an available receive-capable link has positive terminal-suppression capacity, and fragments per packet never exceed 256.
- **LINK-FRG-047**: Defensible inter-fragment timing derives a non-refreshing absolute lifetime; expiry wins at an observation equal to or later than the exposed deadline.
- **LINK-FRG-048**: Intermittent bindings instead use a finite testable retention or progress rule that never retains indefinitely.
- **LINK-FRG-049**: Receive admission enforces every resource bound before retention and classifies capacity discard or eviction as `reassembly-resource-exhausted`.
- **LINK-FRG-050**: Eviction is documented, bounded, link-isolated, test-controllable when randomised, and cannot evict a packet being delivered; mandatory terminal tombstones atomically replace bounded prior tombstone state when necessary.
- **LINK-FRG-051**: Link down discards ending-epoch receive state, abandons and finitely releases unfinished transmit work, and fences all such state from later epochs.
- **LINK-FRG-052**: Retirement releases every residual reassembly resource.
- **LINK-FRG-053**: Receive MTU reduction preserves previously admitted bytes but applies the new bound to new frames without extending context expiry.
- **LINK-FRG-054**: Section 7 defines the ten fragmentation, reassembly, and post-acceptance send classifications.
- **LINK-FRG-055**: Each classification is isolated and does not itself alter availability or unrelated state.
- **LINK-FRG-056** (specification-suite invariant): Section 06 makes every classification observably representable and may aggregate tombstoned repeats.
- **LINK-FRG-057**: Section 8 vectors are normative examples.
- **LINK-FRG-058**: A conformance suite exercises the required packet, frame, ordering, conflict, resource, lifetime, epoch, MTU, destination, and CRC cases with state and classification expectations.
