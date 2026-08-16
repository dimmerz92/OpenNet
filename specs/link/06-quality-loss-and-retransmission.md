# 06. Link Quality, Loss, and Retransmission

## 1. Purpose and boundary

This section defines complete-packet delivery evidence, packet-level retransmission, post-acceptance send outcomes, receive metadata, and bounded diagnostic observability for the Link layer (LINK-QRT-001). It applies to opaque Network packets and does not inspect or assign meaning to their contents.

The default service remains best effort. Reliability, retransmission, quality hints, counters, and acknowledgements in this section are link-local and epoch-scoped and do not provide authentication, confidentiality, durable storage, Network-layer consumption, application delivery, or end-to-end Transport reliability (LINK-QRT-002).

Section 07 owns general ordering and duplicate suppression. It MUST preserve the transaction-scoped suppression required here, but this section does not otherwise pre-empt its mechanisms. Section 08 owns energy policy. Network route choice, rolling quality estimates, and congestion policy remain above this section (LINK-QRT-003).

Requirement IDs in this section use the `LINK-QRT-NNN` prefix.

## 2. Delivery and retransmission capabilities

### 2.1 Best effort and complete-packet reliability

When `reliable_delivery` is absent or false, the Link service provides no delivery proof and MUST NOT infer reliability from medium type, successful local queueing, native transmission completion, per-frame acknowledgement, observed loss, or retransmission behaviour (LINK-QRT-004).

When `reliable_delivery` is true, its unit is one complete opaque packet accepted through one `Send` (LINK-QRT-005). `delivered` means that the remote Link service validated the complete packet, reassembled it where applicable, atomically reserved the state required to make it eligible for one `Receive`, and generated the positive proof defined below. It does not mean that the Network layer consumed or retained the packet.

Version 1 reliable delivery applies only to an eligible unicast or implicit-peer destination (LINK-QRT-006). The eligible set is the stable adapter-profile envelope of every unicast and implicit-peer destination form and native-route class that the binding may make `Send`-admissible while `reliable_delivery` is true. It is independent of transient neighbour presence and current proof availability. Broadcast and multicast remain best effort, do not use packet-level retry or acknowledgement, and cannot produce `delivered`.

A link MAY report `reliable_delivery` only when every eligible destination and route class within its validated capability envelope has a finite proof return association satisfying this section (LINK-QRT-007). The adapter profile MUST enumerate those admissible classes. If any class lacks the required proof association, the capability is false or the reliable subset is exposed through a separate binding. A bidirectional or asymmetric binding may use the canonical acknowledgement. A send-only service binding may use a profile-native equivalent. Loss of a required active proof path is a reliability-affecting coherent change and fails the capability closed. A physically one-way binding without return evidence and every receive-only binding MUST report `reliable_delivery` as false.

### 2.2 Native retransmission

`native_retransmission = true` means that the shim or medium may retry a profile-defined generic-frame, native-packet, or native-subdivision unit without another Link-service submission (LINK-QRT-008). The adapter profile MUST define retry triggers, stopping conditions, possible peer-visible duplication, irreversible commitment, attempt completion, failure, and finite attempt, byte, time, retained-state, and work bounds. Without that complete contract the field is false. A receive-only binding MUST report it as false.

Native retransmission does not imply complete-packet delivery, ordering, duplicate suppression, or authentication; does not change Packet Identifier or `send_id`; and MUST NOT extend the packet transaction's absolute lifetime (LINK-QRT-009). An exact native retry count is diagnostic where available; otherwise it is absent, never estimated.

Every adapter profile used with `reliable_delivery` or `native_retransmission` MUST expose the applicable capability, proof-carriage, commitment, completion, retry, timing, resource, failure, and conformance behaviour before the binding becomes `Available` (LINK-QRT-010). Invalid or incomplete safety-relevant capability input fails closed as absent.

### 2.3 Shim transmission occurrence

Every `Shim-Send` invocation carries a non-zero opaque unsigned 64-bit `shim_tx_id` identifying that exact frame occurrence within its binding and epoch (LINK-QRT-011). It is host-local, never crosses the medium, reveals no packet semantics, and MUST NOT be reused while an outcome, barrier return, cancellation, or delayed callback for the prior occurrence may remain.

After `Shim-Send.accepted`, exactly one ownership-ending path applies to the occurrence (LINK-QRT-012): `Shim-Transmit-Outcome.attempt-complete`, `failed-before-commit`, or `failed-after-commit`; exact return while uncommitted through a selective or size-change barrier; or cancellation through a down barrier. Returned collections preserve `shim_tx_id`. Native retransmission completes before the occurrence outcome. A committed occurrence cannot be returned as uncommitted.

Unknown, duplicate, stale, or late shim outcomes are discarded and counted without revising a terminal packet outcome or affecting unrelated work (LINK-QRT-013). Adapter profiles define the exact commitment and attempt-complete boundaries; the shim need not parse Packet Identifier, `send_id`, fragmentation, addressing, or payload contents.

## 3. Accepted send identity and terminal outcome

### 3.1 Admission and `send_id`

Every successful `Send` returns `accepted` accompanied by a non-zero opaque unsigned 64-bit `send_id` (LINK-QRT-014). Its identity is `(link_id, epoch, send_id)`. It is host-local, never crosses the medium, is unrelated to Packet Identifier, and MUST NOT be reused within an availability epoch.

Before returning `accepted`, the service MUST atomically reserve the `send_id`, immutable packet ownership, framing state, all required bounded attempt or retry state, and capacity for exactly one terminal `Send-Outcome` (LINK-QRT-015). Ordinary capacity exhaustion returns `rejected-queue-full`. The implementation MUST close new send admission and enter a safe lifecycle transition before identifier exhaustion could cause reuse.

### 3.2 `Send-Outcome`

`Send-Outcome` is an upward primitive carrying `link_id`, `epoch`, `send_id`, `result`, and `reason` (LINK-QRT-016). Exactly one immutable outcome MUST be emitted for every accepted send. It cannot be coalesced, shed, duplicated, or revised, and its emission releases all remaining packet ownership and tracking state.

`result` is exactly one of (LINK-QRT-017):

| Result | Meaning |
|---|---|
| `attempted` | Best-effort only: every required local or native action for the complete attempt reached its profile-defined attempt-complete boundary. It asserts no receipt. |
| `delivered` | A matching positive complete-packet proof was accepted for a reliable transaction. |
| `failed` | Complete remote acceptance is proved impossible because no potentially complete attempt crossed commitment and all relevant work was returned or cancelled. |
| `indeterminate` | Complete remote acceptance may have occurred or commitment state is unknown, but no positive proof was accepted. |

For a fixed reliable frame set, possible commitment is cumulative across the complete transaction rather than confined to one retry round. A frame is possibly committed when any of its occurrences reports `attempt-complete` or `failed-after-commit`, or when its commitment state is unknown. A potentially complete packet is possible once every frame in the fixed set has possible committed coverage across any combination of rounds. `failed-before-commit`, exact uncommitted return, and pre-commit cancellation do not contribute. Any unresolved unknown that prevents proving complete remote acceptance impossible requires `indeterminate`.

`reason` is exactly one of `attempt-complete`, `positive-acknowledgement`, `pre-commit-cancelled`, `retry-exhausted`, `transaction-expired`, `destination-invalidated`, `source-claim-rotated`, `size-change-invalidated`, `link-down`, `barrier-failed`, `local-resource-failure`, or `adapter-failure` (LINK-QRT-018). Result describes delivery knowledge; reason describes why tracking ended.

Where several reasons become true at one logical point, the primary reason is selected in this precedence: accepted positive proof; barrier failure; Link-Down; destination invalidation; source-claim rotation; size-change invalidation; transaction expiry; retry exhaustion; adapter failure; local resource failure; pre-commit cancellation; attempt completion (LINK-QRT-019). The result is still derived independently from the commitment evidence in the result table above.

For a single link, `Send-Outcome` is ordered with selective cut-offs, `Link-Changed`, final diagnostics, and `Link-Down`; outcomes from different links may interleave (LINK-QRT-020). Unknown, stale, or duplicate internal callbacks MUST NOT emit another outcome and increment their applicable diagnostic counter.

Best-effort silence is not measured loss (LINK-QRT-021). A delivery ratio has a standard meaning only over transactions eligible for `delivered`; no implementation may report a best-effort packet as delivered merely because its attempt completed.

## 4. Reliable packet identity and fixed frame plan

Packet Identifier is the unsigned 64-bit on-wire identity of a fragmented packet and of every unfragmented reliable packet (LINK-QRT-022). The complete transaction key is `(link_id, epoch, sender_incarnation, Packet Identifier)`. An unfragmented best-effort packet may omit Packet Identifier.

A sender MUST NOT reuse a transaction key while acknowledgement, retry, terminal suppression, delayed input, or native state for the prior use may remain (LINK-QRT-023). Before allocation state could repeat, wrap, or be lost, the sender establishes a new incarnation or fails safely without accepting the packet.

Initial reliable admission fixes one complete bounded generic-frame set (LINK-QRT-024). Every packet-level retry preserves Packet Identifier, source and destination addressing, sender and destination incarnations, fragmentation fields and ranges, Packet CRC-32C, payload bytes, and every other generic-frame byte. Frame transmission order may vary; native timing and subdivision may vary only while reconstructing the same frames.

The transaction reservation retains the immutable frames or bounded reproducible references to them through terminal outcome (LINK-QRT-025). A size, address, source-claim, route, proof-carriage, or capability change that invalidates the fixed plan terminalises the transaction rather than refragmenting it.

Reliable explicit-unicast Data MUST carry Source Link Address and Sender Incarnation, and its destination MUST provide one unambiguous current return route (LINK-QRT-026). Anonymous generic reliable delivery is permitted only on an implicit-peer binding or through a strict profile-native equivalent with an unambiguous return association.

## 5. Canonical packet acknowledgement

### 5.1 Wire form

Version 1 allocates Control Type `0x0002` as `Packet Acknowledgement` (LINK-QRT-027). It is positive acknowledgement only, has an empty body, is never itself acknowledged, and carries no status, reason, Packet Length, Packet CRC-32C, diagnostic text, or payload-derived data.

The explicit-unicast form contains exactly Control Type, Destination Link Address, Destination Incarnation, Source Link Address, Sender Incarnation, and Packet Identifier (LINK-QRT-028). Its destination fields exactly equal the acknowledged packet's source fields, and its source fields exactly equal the acknowledged packet's destination fields. Its maximum header is 123 bytes.

The implicit-peer form contains exactly Control Type and Packet Identifier and omits every section 05 address and incarnation field (LINK-QRT-029). The binding supplies both peer incarnations and the unambiguous reverse association.

Any other field combination, non-empty body, non-local destination, invalid address reversal, absent required return association, or malformed Packet Identifier is `acknowledgement-malformed` and is discarded without changing a transaction or unrelated state (LINK-QRT-030).

### 5.2 Generation and terminal receiver state

The receiver generates a positive acknowledgement only after the complete packet has passed every applicable section 03 through 06 validation and it has atomically reserved `Receive` eligibility, one terminal transaction record, and bounded acknowledgement work (LINK-QRT-031). For a fragmented reliable packet, that terminal record is the authoritative composed completion state and satisfies section 04's completion-tombstone obligation. If any reservation is unavailable, it neither emits `Receive` nor acknowledges and increments `reliable-receive-resource-exhausted`; a later retry may succeed.

Before `Receive` becomes observable, the service installs the terminal record keyed by link, epoch, sender incarnation, and Packet Identifier (LINK-QRT-032). It retains no packet body after delivery, only bounded identity, address, route, expiry, and acknowledgement state sufficient to suppress another `Receive` and repeat acknowledgement.

The terminal record has a fixed absolute expiry no earlier than the adapter profile's maximum reliable transaction lifetime plus maximum delayed native-input allowance (LINK-QRT-033). Retry and acknowledgement do not refresh it. A live record MUST NOT be evicted to admit another transaction.

A matching retry while its terminal record is live produces no additional `Receive`, allocates no reassembly context or new terminal record, retains no fragment body, and does not refresh expiry (LINK-QRT-034). For a fragmented transaction, the first structurally and semantically valid fragment matching the full retained transaction identity is sufficient to recognise the retry and provide one bounded acknowledgement opportunity under the profile timing and output limits. Malformed, expired, or identity-mismatching input cannot trigger it. After the acknowledgement-transmission budget is exhausted, the record still suppresses duplicate delivery until expiry.

### 5.3 Sender validation

A received canonical or native-equivalent proof yields `delivered` only when it matches one current pending reliable transaction's link, epoch, sender incarnation, Packet Identifier, destination, source claim, and proof-carriage association (LINK-QRT-035). Matching has no effect on another transaction.

An unknown, stale, duplicate, wrong-epoch, non-local, malformed, or inconsistent proof is discarded and increments the inclusive `acknowledgements-invalid` activity counter (LINK-QRT-036). A malformed proof also increments the narrower `acknowledgement-malformed` classification counter at the same observation point. Other structurally well-formed invalid proofs increment only the inclusive counter unless another fixed classification applies. A proof received exactly at or after transaction expiry loses to expiry. A valid proof logically before a cut-off wins under the terminal-reason precedence above.

An adapter profile MAY replace the generic acknowledgement carriage only when the receiving Link service triggers the native proof after the same complete-packet reservation boundary and the profile provides equivalent transaction identity, ordering, retry, loss, bounds, malformed-input handling, and conformance evidence (LINK-QRT-037). Local queueing, transmission completion, or per-frame native acknowledgement is insufficient.

## 6. Whole-packet retransmission and timing

The canonical v1 reliability mechanism retries the complete fixed frame set; it defines no selective fragment acknowledgement (LINK-QRT-038). A future extension may select a subset of the fixed set but cannot redefine live fragment boundaries.

Every reliable adapter profile MUST define finite values for maximum complete-packet forward allowance, receiver validation and reassembly delay, acknowledgement generation delay, acknowledgement return allowance, sender acknowledgement wait, minimum and maximum retry delay, packet-level attempt count, absolute transaction lifetime, delayed native-input allowance, receiver terminal-record lifetime, acknowledgement transmissions per record, acknowledgement suppression interval, retained bytes, and combined native and packet retry work (LINK-QRT-039).

The sender acknowledgement wait MUST be greater than the sum of maximum complete-packet forward allowance, receiver validation and reassembly delay, acknowledgement generation delay, and acknowledgement return allowance (LINK-QRT-040). Terminal-record lifetime MUST be at least maximum transaction lifetime plus maximum delayed native-input allowance. Values use relative local time and require no shared clock.

At a logical time equal to a deadline, expiry wins over retry, acknowledgement, refresh, or new work (LINK-QRT-041). Retry never refreshes the absolute transaction deadline and MUST NOT begin when the attempt cannot complete within the remaining lifetime.

Acknowledgement repetition MAY be rate-limited, but the profile MUST retain at least one response opportunity compatible with the sender retry schedule (LINK-QRT-042). A malformed or replayed input MUST NOT produce unbounded acknowledgement work or refresh terminal state.

Exhausting retry or lifetime bounds after a potentially complete committed attempt yields `indeterminate` (LINK-QRT-043). A locally established condition proving that no potentially complete attempt committed may yield `failed`. Version 1 defines no negative acknowledgement.

## 7. Changes, cut-offs, and lifecycle

A destination or route invalidation and a source-claim rotation apply the selective cut-offs defined in sections 02 and 05 and terminalise each affected packet once (LINK-QRT-044). Prior positive proof is `delivered`; completed best effort is `attempted`; no potentially complete commitment is `failed`; and possible or unknown commitment without proof is `indeterminate`, with the applicable reason.

`Link-Changed` terminalises only transactions affected by a change to framing, destination validity, reliable-delivery semantics, proof carriage, native retransmission, retry bounds, or transaction interpretation (LINK-QRT-045). Unaffected work continues. An added capability does not upgrade a transaction accepted under a prior tuple, and informational quality-hint change alone does not disturb work.

Returned uncommitted frames of a terminalised transaction are cancelled and MUST NOT be reframed or resubmitted (LINK-QRT-046). New admission begins only under the authoritative new tuple. A fixed frame set still valid under the new tuple may continue only when none of the affected semantics named above changed.

Before `Link-Down`, the service closes admission, establishes shim and receive cut-offs, processes valid acknowledgements logically before the cut-off, terminalises and emits every pending `Send-Outcome`, finalises neighbour state, and classifies every admitted receive that is delivered or discarded (LINK-QRT-047). It then establishes a diagnostic update cut-off, emits `Link-Diagnostics-Final`, and emits `Link-Down`. At the diagnostic cut-off, every required ending transition and accepted pre-cut-off input has been processed or fenced and the ending epoch's counters become immutable. Later old-epoch callbacks, proofs, retries, metadata, and diagnostic notifications are discarded without mutating the final snapshot. A bounded non-normative implementation diagnostic may record such callbacks but cannot become current protocol state.

At the down cut-off, prior positive proof is `delivered`; completed best effort is `attempted`; no potentially complete commitment is `failed` with `link-down`; possible commitment without proof is `indeterminate` with `link-down`; and unknown commitment after barrier failure is `indeterminate` with `barrier-failed` (LINK-QRT-048). A native transmission permitted to complete physically later cannot revise the outcome.

Transition to `Retired` releases every remaining section 06 transaction, terminal record, metadata item, counter extension, timer, reservation, queue, callback, and diagnostic resource (LINK-QRT-049).

## 8. Receive metadata

`recv_meta` is an optional bounded host-local collection attached to one `Receive` (LINK-QRT-050). It may be empty. Every field has exactly one provenance of `shim-reported` or `link-service-derived`; all values are unauthenticated and advisory and never alter generic-frame bytes.

The standard optional time fields are `observed_at_ns`, an unsigned 64-bit value from an implementation-defined host-local monotonic origin, and `time_resolution_ns`, a positive unsigned 64-bit resolution (LINK-QRT-051). The Link service supplies or safely normalises the time at the reception's logical observation point. It is not wall-clock time and is not comparable across nodes or host restarts. Unsafe conversion, wrap, or discontinuity omits the standard value rather than moving backwards.

The standard optional physical fields are (LINK-QRT-052):

| Field | Type and unit |
|---|---|
| `signal_power_centidbm` | signed 16-bit hundredths of a dBm |
| `signal_power_resolution_centidbm` | positive unsigned 16-bit hundredths of a dBm |
| `signal_power_saturated` | boolean |
| `snr_centidb` | signed 16-bit hundredths of a dB |
| `snr_resolution_centidb` | positive unsigned 16-bit hundredths of a dB |
| `snr_saturated` | boolean |

A physical field is present only when the shim genuinely supplies that physical unit (LINK-QRT-053). An arbitrary RSSI, percentage, rank, or vendor score cannot populate it. Missing means unavailable. Out-of-range input is explicitly saturated or omitted, never wrapped. Resolution and representation precision do not assert measurement accuracy.

Each standard metadata field occurs at most once (LINK-QRT-054). A malformed, duplicate, unknown, or excess field is dropped or normalised independently and increments `receive-metadata-invalid`; it MUST NOT discard an otherwise valid packet or another valid metadata field.

Adapter-profile metadata extensions MUST define a bounded identifier, exact type, provenance, units or opaque semantics, direction, range, unavailable handling, saturation behaviour, and finite entry, byte, depth, and work limits (LINK-QRT-055). Unknown extensions are ignored within the global metadata budget.

Metadata is immutable for its `Receive`, cannot retroactively revise another reception or diagnostic state, and MUST NOT establish authentication, identity, lifecycle truth, protocol correctness, or a universal cross-medium quality score (LINK-QRT-056).

## 9. Diagnostic observability

### 9.1 Counter representation and catalogue

Each standard diagnostic counter is an unsigned 64-bit value scoped to one link and availability epoch, initialised to zero, monotonically incremented, and saturated at `2^64 - 1` without wrapping or decrementing (LINK-QRT-057). Each counter carries its own `saturated` boolean. Saturation of one counter does not stop another.

The standard activity counters are `packets-accepted`, `outcome-attempted`, `outcome-delivered`, `outcome-failed`, `outcome-indeterminate`, `packet-retry-attempts`, `native-retry-attempts`, `packets-admitted-to-receive`, `valid-receive-resource-discards`, `valid-receive-lifecycle-discards`, `acknowledgements-generated`, `acknowledgements-repeated`, `acknowledgements-suppressed`, `acknowledgements-invalid`, `receive-metadata-invalid`, `diagnostic-notifications-coalesced`, and `internal-callback-invalid` (LINK-QRT-058). `valid-receive-lifecycle-discards` increments when an otherwise valid reception admitted before the ending receive cut-off is prevented from becoming `Receive` by that lifecycle cut-off. `native-retry-attempts` is accompanied by `unavailable` when exact reporting is absent.

The standard classification counters include `receive-frame-size-exceeded`, `malformed-frame`, `stream-record-timeout`, every classification in LINK-FRG-054, and `addressing-malformed`, `destination-not-local`, `destination-incarnation-mismatch`, `neighbour-resource-exhausted`, `send-destination-invalidated`, `source-claim-rotated`, `acknowledgement-malformed`, and `reliable-receive-resource-exhausted` (LINK-QRT-059). Each terminal outcome also increments exactly one result counter and one reason counter. The twelve reason-counter identifiers are `outcome-reason-attempt-complete`, `outcome-reason-positive-acknowledgement`, `outcome-reason-pre-commit-cancelled`, `outcome-reason-retry-exhausted`, `outcome-reason-transaction-expired`, `outcome-reason-destination-invalidated`, `outcome-reason-source-claim-rotated`, `outcome-reason-size-change-invalidated`, `outcome-reason-link-down`, `outcome-reason-barrier-failed`, `outcome-reason-local-resource-failure`, and `outcome-reason-adapter-failure`. Every standard result and reason counter, including a zero-valued counter and its saturation flag, is present from epoch initialisation.

Counter update and its classified state transition share one logical observation point (LINK-QRT-060). A same-epoch delta is meaningful only when relevant counters are unsaturated. Cross-epoch subtraction has no standard meaning. Best-effort silence and unavailable native retry counts do not increment a loss counter.

Standard counter identifiers are fixed and cannot contain addresses, packet identifiers, payload-derived labels, free text, or peer-created cardinality (LINK-QRT-061). Profile extensions use bounded identifiers and the same value, saturation, and optional unavailable form. Unknown extensions are ignored.

### 9.2 Change notification and query

`Link-Diagnostics-Changed` is a coalescible upward notification carrying only `link_id` and `epoch` (LINK-QRT-062). At most one pending occurrence per consumer and link is required. When that occurrence first absorbs one or more otherwise-notifiable changes, `diagnostic-notifications-coalesced` increments exactly once for the pending episode. The increment is covered by the already pending notification and cannot itself cause another increment. A new episode begins only after the pending notification becomes observable. Authoritative counters preserve the available aggregate information.

`Query-Link-Diagnostics` is a downward primitive taking `link_id` and `epoch` and returning exactly one of `current`, `rejected-link-unavailable`, `rejected-stale-epoch`, or `rejected-malformed` (LINK-QRT-063). Recognition and compound rejection precedence are identical to `Enumerate-Neighbours`. `current` contains every standard counter and flag plus bounded extensions at one logical observation point and has no side effects.

For the same consumer and link, diagnostic notification, query, send outcome, neighbour event, `Receive`, `Link-Changed`, and lifecycle observation points are mutually ordered (LINK-QRT-064). A query observes every logically prior counter change and none logically later.

### 9.3 Final epoch snapshot

Every exposed availability epoch reserves capacity for exactly one `Link-Diagnostics-Final` event carrying `link_id`, ending `epoch`, every standard counter and flag, and bounded extensions (LINK-QRT-065). It is authoritative, non-coalescible, non-sheddable, and emitted after the diagnostic update cut-off in the lifecycle order defined above. Its snapshot is immutable and includes every required ending-epoch update logically before that cut-off and none after it. It contains no packet identity, address, payload data, or free text.

`Query-Link-Diagnostics` remains current-epoch only (LINK-QRT-066). While a recognised unavailable or retired link record remains retained, `Link-Query` MAY include the most recent final diagnostic snapshot as stale informational state. A later epoch starts zeroed counters and cannot reorder or overwrite the prior final event. No historical retention is required after record release.

## 10. Resource, congestion, and security requirements

Each implementation MUST configure and expose finite per-link and global limits for accepted packets, pending outcomes, retained packet and frame bytes, transaction and terminal records, packet and native attempts, acknowledgement state and output, timers, metadata entries and bytes, counters and extensions, diagnostic snapshots and notifications, callback state, scheduler work, and total processing work (LINK-QRT-067).

`Send` reservations protect completion of already accepted work from later admission (LINK-QRT-068). Retries remain within the transaction's reservation and combined work budget. Queue pressure returns `rejected-queue-full` and updates diagnostics; it does not by itself change MTU, `max_packet_size`, descriptor state, or link availability.

Scheduling among Data, Control, receives, new sends, retries, and acknowledgements is implementation-defined but MUST use bounded work per decision, preserve cross-link resource isolation, and provide documented finite starvation bounds for accepted work and required acknowledgement opportunities (LINK-QRT-069). Version 1 mandates no universal Control priority, congestion window, pacing algorithm, or end-to-end rate protocol.

Acknowledgements, receive metadata, quality hints, counters derived from hostile traffic, and native delivery reports are unauthenticated (LINK-QRT-070). They may be forged, replayed, delayed, suppressed, or used to exhaust bounded state. A matching positive acknowledgement is operational Link evidence only and MUST NOT bypass higher-layer authentication or authorisation.

Malformed or hostile input on one transaction or link MUST NOT allocate unbounded state, refresh unrelated state, change another link, create an outcome for another packet, or make a link unavailable by itself (LINK-QRT-071). A receiver MUST bound acknowledgement amplification by its profile timing, output, and terminal-record limits.

No implementation may synthesise a universal quality score from the fields in this section as a Link conformance result (LINK-QRT-072). The Network layer may derive policy, ratios, windows, or rankings, but cannot depend on optional quality hints for protocol correctness.

## 11. Conformance cases

A conformance suite MUST exercise wire bytes, terminal results and reasons, ownership, counters, metadata, resource effects, timing boundaries, and unrelated-state isolation for at least (LINK-QRT-073):

- best-effort unicast, implicit, broadcast, and multicast attempts;
- reliable unfragmented and fragmented packets with matching acknowledgement;
- acknowledgement loss, retry, duplicate acknowledgement, stale acknowledgement, forgery, replay, malformed addressing, and wrong epoch;
- fixed-frame retries in different frame orders, cumulative mixed-round commitment coverage, definite no-commit evidence, unknown commitment, and prohibited refragmentation;
- receiver terminal-record capacity, composition with fragmented completion state, non-eviction, exact expiry, first valid partial retry recognition, malformed and identity-mismatching retry fragments, repeated acknowledgement, and output exhaustion;
- every result and reason, including compound reason precedence and unknown commitment;
- retry, transaction, delayed-input, and exact-deadline boundaries;
- canonical and strict native-equivalent proof on bidirectional, asymmetric, send-only, and physically one-way profiles;
- destination, route, source-claim, size, reliability, barrier, and Link-Down cut-offs before and after commitment;
- metadata absence, valid limits, duplicates, malformed values, saturation, unknown extensions, clock discontinuity, and packet isolation;
- every standard counter and exact identifier, resource and lifecycle receive discards, inclusive invalid-acknowledgement accounting, saturation, unavailable native count, non-recursive notification coalescing, current query, diagnostic update cut-off, immutable final snapshot, epoch reset, and retained stale `Link-Query` view;
- queue, byte, attempt, callback, outcome, acknowledgement, metadata, diagnostic, scheduler, and global exhaustion; and
- hostile activity on one packet, neighbour, transaction, link, or consumer without unrelated effects.

## 12. Implementation freedom

Implementations may choose retry scheduling, timer structures, counter storage, event queues, packet retention, acknowledgement scheduling, metadata representation, diagnostic extension storage, fairness algorithms, and native retransmission mechanisms (LINK-QRT-074). They conform only when the wire form, capability truth, proof boundary, terminal outcome, commitment classification, fixed frame plan, timing inequalities, bounds, lifecycle order, diagnostics, metadata semantics, security limits, and observable conformance behaviour match this section.

## 13. Requirement summary

This summary is a navigation aid. The normative prose above is authoritative.

- **LINK-QRT-001**: Section 06 defines complete-packet delivery evidence, retransmission, send outcomes, receive metadata, and diagnostic observability.
- **LINK-QRT-002**: Section 06 mechanisms are link-local and provide no authentication, durable storage, upper-layer consumption, or end-to-end delivery.
- **LINK-QRT-003**: Later sections preserve transaction suppression; ordering, general duplication, energy, route policy, and rolling estimates remain outside section 06.
- **LINK-QRT-004**: Best effort infers no reliability from medium, queueing, native completion, per-frame acknowledgement, loss, or retry.
- **LINK-QRT-005**: Reliable delivery proves complete remote Link acceptance and `Receive` eligibility for one accepted packet.
- **LINK-QRT-006**: Version 1 reliability applies to the stable profile-enumerated envelope of admissible unicast and implicit-peer destination and route classes.
- **LINK-QRT-007**: Every eligible class requires a finite proof association or a separate binding; capability loss fails closed, and receive-only and proof-less one-way bindings report false.
- **LINK-QRT-008**: Native retransmission is a fully bounded profile-defined retry mechanism below Link packet submission.
- **LINK-QRT-009**: Native retry implies no delivery, ordering, suppression, or authentication and cannot change identity or lifetime.
- **LINK-QRT-010**: Profiles expose complete capability, proof, retry, bound, and failure contracts before availability or fail closed.
- **LINK-QRT-011**: Every shim frame occurrence has one non-reused epoch-scoped host-local 64-bit `shim_tx_id` carrying no packet semantics.
- **LINK-QRT-012**: An accepted shim occurrence ends exactly once by outcome, uncommitted return, or down cancellation, preserving its identifier.
- **LINK-QRT-013**: Invalid shim outcomes are isolated and counted; profiles define commitment and completion without requiring semantic parsing.
- **LINK-QRT-014**: Every accepted Send returns a non-zero epoch-scoped host-local 64-bit `send_id` not reused within the epoch.
- **LINK-QRT-015**: Acceptance atomically reserves identity, ownership, framing, retry, and one exact outcome capacity.
- **LINK-QRT-016**: Every accepted send emits one immutable non-sheddable `Send-Outcome` and releases ownership.
- **LINK-QRT-017**: Terminal result is exactly `attempted`, `delivered`, `failed`, or `indeterminate`, with cumulative possible commitment across the fixed frame set and all retry rounds.
- **LINK-QRT-018**: Terminal reason is exactly one of the twelve standard reasons and is distinct from result.
- **LINK-QRT-019**: Compound terminal reasons use the defined precedence while result remains commitment-derived.
- **LINK-QRT-020**: Outcomes are link-locally ordered with cut-offs and lifecycle; invalid internal callbacks cannot duplicate them.
- **LINK-QRT-021**: Best-effort silence is not loss and attempt completion is not delivery.
- **LINK-QRT-022**: Packet Identifier identifies fragmented and unfragmented reliable packets; unfragmented best effort may omit it.
- **LINK-QRT-023**: A transaction key is not reused while any related proof, retry, suppression, delayed, or native state may remain.
- **LINK-QRT-024**: Reliable admission fixes one byte-identical generic-frame set for every packet-level retry.
- **LINK-QRT-025**: Transactions retain bounded immutable frame state and terminalise rather than refragment across invalidating changes.
- **LINK-QRT-026**: Explicit reliable Data carries source and return-route identity; anonymous generic reliability requires an implicit or native association.
- **LINK-QRT-027**: Control Type `0x0002` is positive-only empty-body Packet Acknowledgement.
- **LINK-QRT-028**: Explicit acknowledgement reverses source and destination claims, includes Packet Identifier, and has maximum header 123.
- **LINK-QRT-029**: Implicit acknowledgement contains Control Type and Packet Identifier only.
- **LINK-QRT-030**: Invalid acknowledgement wire or addressing form is isolated `acknowledgement-malformed`.
- **LINK-QRT-031**: A receiver acknowledges only after complete validation and atomic reservation of Receive, composed terminal completion, and acknowledgement capacity.
- **LINK-QRT-032**: Terminal state is installed before Receive and retains only bounded suppression and proof metadata after delivery.
- **LINK-QRT-033**: Terminal lifetime covers transaction plus delayed input, never refreshes, and live records are non-evictable.
- **LINK-QRT-034**: The first valid matching fragment of a live reliable retry produces no Receive or reassembly state and only another bounded acknowledgement opportunity.
- **LINK-QRT-035**: Delivered requires a proof matching every current pending transaction component and association.
- **LINK-QRT-036**: Invalid proofs increment an inclusive activity counter, malformed proofs also increment their classification, and exact-deadline expiry wins.
- **LINK-QRT-037**: Native proof carriage is permitted only at the identical complete-packet boundary with equivalent observable semantics.
- **LINK-QRT-038**: Canonical v1 retries the complete fixed frame set and defines no selective fragment acknowledgement.
- **LINK-QRT-039**: Reliable profiles define every finite timing, attempt, state, output, byte, and work bound.
- **LINK-QRT-040**: Acknowledgement wait and terminal lifetime satisfy the defined worst-case inequalities without a shared clock.
- **LINK-QRT-041**: Exact-deadline expiry wins, deadlines never refresh, and impossible-to-finish attempts do not start.
- **LINK-QRT-042**: Acknowledgement rate limiting remains compatible with at least one sender response opportunity and cannot be attacker-refreshed.
- **LINK-QRT-043**: Retry or lifetime exhaustion after possible commitment is indeterminate; definite no-commit evidence may fail; v1 has no negative acknowledgement.
- **LINK-QRT-044**: Destination, route, and claim cut-offs terminalise each affected packet once from proof and commitment evidence.
- **LINK-QRT-045**: Link changes terminalise only semantically affected transactions and never upgrade old work.
- **LINK-QRT-046**: Terminalised returned work is cancelled rather than reframed; new work waits for the new authoritative tuple.
- **LINK-QRT-047**: Link-Down follows exact cut-offs and ending transitions, then freezes counters at a diagnostic update cut-off before final diagnostics and down observation.
- **LINK-QRT-048**: Down and barrier outcomes use the defined proof and commitment mapping and cannot be revised later.
- **LINK-QRT-049**: Retirement releases every remaining section 06 resource.
- **LINK-QRT-050**: `recv_meta` is one optional bounded immutable unauthenticated host-local collection with explicit provenance.
- **LINK-QRT-051**: Standard observation time is optional host-monotonic nanoseconds with safe resolution and discontinuity behaviour.
- **LINK-QRT-052**: Standard physical signal power and SNR fields use the defined signed centi-unit values, resolutions, and saturation flags.
- **LINK-QRT-053**: Physical fields require genuine units; missing is unavailable and values saturate or omit, never wrap.
- **LINK-QRT-054**: Invalid, duplicate, unknown, or excess metadata is independently dropped and counted without packet loss.
- **LINK-QRT-055**: Metadata extensions define exact bounded identifiers, types, provenance, semantics, and limits.
- **LINK-QRT-056**: Metadata is reception-local advisory state and establishes no identity, lifecycle, correctness, or universal quality.
- **LINK-QRT-057**: Diagnostic counters are epoch-scoped monotonic saturating unsigned 64-bit values with independent flags.
- **LINK-QRT-058**: Fixed activity counters distinguish resource and lifecycle receive discards and define exact native retry unavailable state.
- **LINK-QRT-059**: Fixed classification, result, and twelve exact reason-counter identifiers cover sections 03 through 06.
- **LINK-QRT-060**: Counters update with their state transitions; saturated and cross-epoch deltas have limited meaning and silence is not loss.
- **LINK-QRT-061**: Counter identifiers have bounded fixed cardinality and extensions cannot contain attacker-created keys.
- **LINK-QRT-062**: `Link-Diagnostics-Changed` coalesces once per pending episode without recursively counting its own counter update.
- **LINK-QRT-063**: `Query-Link-Diagnostics` applies defined rejection precedence and returns one side-effect-free authoritative snapshot.
- **LINK-QRT-064**: Diagnostics are mutually ordered with events, Receive, changes, outcomes, and lifecycle for each consumer and link.
- **LINK-QRT-065**: Each epoch reserves and emits one immutable non-sheddable final snapshot after the diagnostic update cut-off.
- **LINK-QRT-066**: Diagnostic query is current-only; retained Link-Query may show stale final state; later epochs start zeroed and ordered.
- **LINK-QRT-067**: Every transaction, outcome, retry, acknowledgement, metadata, diagnostic, callback, timer, byte, and work resource is finitely bounded.
- **LINK-QRT-068**: Accepted reservations are protected; queue pressure rejects and counts without altering capacity or availability.
- **LINK-QRT-069**: Scheduling is policy-free but bounded, isolated, and subject to finite accepted-work and acknowledgement starvation limits.
- **LINK-QRT-070**: Proof, metadata, hints, and hostile-input counters are unauthenticated and cannot replace higher-layer security.
- **LINK-QRT-071**: Hostile input remains bounded and isolated and acknowledgement amplification is profile-limited.
- **LINK-QRT-072**: Link conformance defines no universal quality score; derived policy remains above Link.
- **LINK-QRT-073**: Conformance exercises the required delivery, retry, wire, metadata, diagnostic, resource, lifecycle, and hostile-input cases.
- **LINK-QRT-074**: Internal retry, storage, timing, scheduling, metadata, and diagnostic mechanisms remain free behind observable invariants.
