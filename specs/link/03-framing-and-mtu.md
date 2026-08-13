# 03. Framing and MTU

## 1. Purpose

This section defines the canonical generic Link frame, maximum transmission unit (MTU), directional frame-size enforcement, and canonical adaptation profile for a bare byte stream. Every medium carries the same generic frame across the link-service-to-shim boundary; a shim may map that frame to native units but MUST reconstruct the byte-identical complete frame on receipt (LINK-FRM-001).

This section defines framing only. Section 04 defines fragmentation, reassembly, and maximum Network packet size. Section 05 defines link-layer addressing. Section 06 defines reliability and the standard form of discard observability. Later sections may define fields and control messages within the extension points explicitly reserved here, but they do not change this section's framing rules.

Requirement IDs in this section use the `LINK-FRM-NNN` prefix.

## 2. Generic frame model

### 2.1 Complete-frame boundary

One generic frame is one complete byte sequence passed to `Shim-Send` or emitted by `Shim-Receive` (LINK-FRM-002). Partial generic frames MUST NOT cross the shim boundary. The complete sequence length is supplied by that boundary or by its adapter profile and is not repeated inside the generic frame.

The shim treats a generic frame as opaque bytes. It MAY subdivide, aggregate, delimit, pad, or otherwise map those bytes to native units as permitted by section 02, but it MUST NOT expose a partial reconstruction or alter any generic-frame byte.

### 2.2 Frame layout

A version 1 generic frame has this layout (LINK-FRM-003):

```text
+---------+-------------+-----------------+
| Version | Frame class | Header length   |
| 1 byte  | 1 byte      | 1 byte          |
+---------+-------------+-----------------+
| Variable header: ordered TLVs           |
+-----------------------------------------+
| Body: payload or control body           |
+-----------------------------------------+
| Optional explicit Link padding          |
+-----------------------------------------+
```

The three-byte fixed header fields are:

- `version`: `0x01` for this specification. `0x00` is invalid. A receiver MUST discard a frame carrying an unsupported version without interpreting its variable header or body (LINK-FRM-004).
- `frame_class`: `0x01` for `Data` or `0x02` for `Control`. `0x00` and every value not defined for the received version are invalid. A receiver MUST discard an unknown frame class without interpreting its variable header or body (LINK-FRM-005).
- `header_length`: the total number of bytes in the fixed and variable headers. It MUST be at least 3, MUST be no greater than 255, and MUST be no greater than the complete frame length (LINK-FRM-006).

A `Data` frame carries either one complete opaque Network packet or one fragment of such a packet. Its body MUST contain at least one byte (LINK-FRM-007). Section 04 defines how fragmentation is identified and processed.

A `Control` frame carries Link-owned protocol information. Its variable header MUST identify the control message, and its body MAY be empty only when the definition of that control message permits an empty body (LINK-FRM-008).

### 2.3 Type-Length-Value fields

The variable header is a sequence of Type-Length-Value fields, abbreviated TLVs. Each TLV consists of a one-byte `type`, a one-byte `length`, and exactly `length` value bytes (LINK-FRM-009). A zero value length is permitted only when the definition of that TLV type explicitly permits it.

TLVs MUST occur in strictly increasing numeric `type` order (LINK-FRM-010). Consequently, the same type cannot occur twice in one frame. A duplicated, reordered, truncated, or overlong TLV makes the frame malformed.

A receiver MUST recognise every TLV in a version 1 frame and MUST verify that each TLV is permitted for the selected frame class and has a length and value allowed by its definition (LINK-FRM-011). An unknown, reserved, class-forbidden, or incorrectly encoded TLV makes the frame malformed. Version 1 does not define a generic rule for silently ignoring unknown fields because an unknown field may affect delivery or security semantics.

The version 1 TLV namespace begins with these assignments (LINK-FRM-012):

| Type | Name | Allowed class | Value length |
|---:|---|---|---:|
| `0x00` | Invalid | none | none |
| `0x01` | Control Type | Control | 2 |
| `0x02` to `0xFE` | Allocated by later Link sections | as defined | as defined |
| `0xFF` | Padding Length | Data or Control | 2 |

Every value from `0x02` through `0xFE` that is allocated by the version 1 Link Specification MUST have exactly one meaning across all Link sections (LINK-FRM-013). Unallocated values remain unknown and MUST NOT be transmitted. This is a specification-suite invariant and does not establish a runtime allocation service.

### 2.4 Control Type

Every `Control` frame MUST contain exactly one Control Type TLV, and a `Data` frame MUST NOT contain it (LINK-FRM-014). Its two-byte value is an unsigned big-endian control-message identifier. This section defines the dispatch field but allocates no control-message identifiers. The section that allocates an identifier defines its permitted TLVs, body syntax, body-size limits, and whether its body may be empty. An unknown Control Type makes the frame malformed.

Every Control Type allocated by the version 1 Link Specification MUST have exactly one meaning across all Link sections (LINK-FRM-015). An unallocated Control Type MUST NOT be transmitted. This is a specification-suite invariant and does not establish a runtime allocation service.

### 2.5 Explicit Link padding

The optional Padding Length TLV contains an unsigned 16-bit big-endian count of padding bytes at the end of the complete frame (LINK-FRM-016). Absence means zero padding. A present value of zero is non-canonical and makes the frame malformed. A padding count greater than the bytes remaining after the header also makes the frame malformed.

A sender MUST set every explicit Link padding byte to `0x00` and MUST place padding only at the end of the frame (LINK-FRM-017). A receiver MUST reject a frame containing a non-zero explicit Link padding byte; otherwise it removes the padding before passing the body to class-specific processing. Link padding is not authentication and is not a traffic-analysis defence. Native-medium-required padding remains shim-owned and does not form part of the generic frame. End-to-end privacy padding remains the responsibility of Transport.

The body length is derived without another wire field (LINK-FRM-018):

```text
body_length = complete_frame_length - header_length - padding_length
```

Later sections MUST express body-affecting information through fields allowed by this format and MUST NOT make the body boundary depend on an implementation-specific representation (LINK-FRM-019).

## 3. MTU and frame capacity

### 3.1 Global bounds

The guaranteed minimum MTU is 256 bytes (LINK-FRM-020). Every compliant available link supports a generic frame of at least that total size, although an individual transmitted frame may be smaller.

The global maximum generic frame size is 65,535 bytes (LINK-FRM-021). No descriptor, adapter profile, generic frame, or reassembled generic-frame representation may exceed this bound.

Every operative send or receive frame-size value MUST be an integer from 256 through 65,535 inclusive (LINK-FRM-022), restating the descriptor validation requirements of section 02.

### 3.2 Directional limits

The operative limits are (LINK-FRM-023):

| Link direction | Send limit | Receive limit |
|---|---|---|
| `bidirectional` | `maximum_frame_size` | defaulted `receive_max_frame_size` |
| `asymmetric` | `maximum_frame_size` | defaulted `receive_max_frame_size` |
| `send-only` | `maximum_frame_size` | receive not permitted |
| `receive-only` | send not permitted | `maximum_frame_size` |

When present on a receive-capable link, `receive_max_frame_size` is informational to the Network layer but normative inside the Link layer. Its defaulting and validation are defined in section 02.

### 3.3 Capacity and maximum packet size

For a candidate frame whose required fields are known, its maximum body capacity is (LINK-FRM-024):

```text
maximum_body_capacity = operative_send_MTU - header_length - padding_length
```

The Link service MUST account for the actual header and explicit Link padding of each frame. A later section MUST NOT define a mandatory frame shape whose header cannot fit within the guaranteed minimum MTU.

Maximum packet size is not the capacity of one frame. It is the largest opaque Network payload accepted by one `Send` invocation and may exceed the MTU through Link fragmentation (LINK-FRM-025). Section 04 defines it from the capacities established here and from its fragmentation and reassembly bounds.

### 3.4 Send enforcement

Before passing a generic frame to `Shim-Send`, the Link service MUST verify that its complete length does not exceed the current operative send MTU and that it passes the structural rules of this section (LINK-FRM-026). An oversized or malformed frame MUST NOT reach native transmission.

The shim MUST independently reject as `rejected-malformed` any complete frame presented to `Shim-Send` that exceeds the current `maximum_frame_size` or intrinsically violates the applicable adapter profile's admission rules (LINK-FRM-027), restating LINK-MED-068 and LINK-MED-107. Transient exhaustion of bounded shim capacity MUST return `rejected-queue-full`. Full generic-frame structural and semantic validation belongs to the Link service; the shim need not parse TLVs, Control Types, or body semantics. MTU reductions and ownership of committed and uncommitted work follow the size-change barrier in sections 01 and 02.

### 3.5 Receive enforcement

On a receive-capable link, the shim MUST enforce the operative receive limit while reconstructing a generic frame, and the Link service MUST independently check the completed length before parsing the fixed header (LINK-FRM-028), restating LINK-MED-106. Neither boundary may allocate or retain frame storage beyond its applicable adapter and frame bounds.

An otherwise complete reception exceeding the operative receive limit MUST be discarded in isolation with classification `receive-frame-size-exceeded` (LINK-FRM-029). This discard MUST NOT change link availability or affect another frame, flow, or link. It MUST be observable through the standard per-link discard mechanism defined in section 06.

## 4. Canonical validation

### 4.1 Validation order

A receiver MUST validate a completed generic frame in this order (LINK-FRM-030):

1. Check the operative receive limit and global maximum.
2. Require at least three bytes before reading the fixed header.
3. Validate `version` and `frame_class`.
4. Validate `header_length` against the complete frame length.
5. Parse TLVs only from byte offset 3 up to, but not including, `header_length`.
6. For each TLV, require two bytes for Type and Length, require the stated value to end within the header, and enforce strict Type ordering.
7. Validate field recognition, field length, field value, and permission for the frame class.
8. Require every field mandatory for the frame class and selected control message.
9. Validate Padding Length and derive the body bounds.
10. Apply the class-specific body rule.
11. Pass the validated frame to the later mechanism that owns its semantics.

Every validation failure not assigned a more specific standard classification is `malformed-frame` and causes only that frame to be discarded (LINK-FRM-031). A malformed frame MUST NOT alter unrelated protocol state or make the link unavailable solely because it is malformed.

No variable header, body, padding, fragmentation, addressing, or control semantic may be acted upon before the preceding structural bounds needed to access it have passed (LINK-FRM-032). Validation MAY be implemented incrementally if its externally observable result is identical to the order above.

An implementation MUST derive allocation and work bounds from validated lengths and MUST NOT trust an unvalidated frame or TLV length for allocation, copying, iteration, or pointer arithmetic (LINK-FRM-033).

## 5. Canonical bare-stream profile

### 5.1 Applicability and record layout

A byte stream without an established native record and recovery convention satisfying section 02 MUST carry generic frames using the canonical profile in this section (LINK-FRM-034), restating LINK-MED-066.

One canonical stream record is (LINK-FRM-035):

```text
0x00 || COBS(envelope_content) || 0x00

envelope_content =
    envelope_version       1 byte
    frame_length           2 bytes
    generic_frame          frame_length bytes
    crc_32c                4 bytes
```

Adjacent records MAY share their boundary delimiter, producing `0x00 encoded-record 0x00 encoded-record 0x00` (LINK-FRM-036).

The envelope version is `0x01`; `0x00` and unsupported values are invalid (LINK-FRM-037). Envelope versioning is independent of generic-frame versioning.

`frame_length` is an unsigned 16-bit big-endian integer counting only generic-frame bytes (LINK-FRM-038). It MUST equal the decoded bytes between the length field and CRC, MUST be at least 3, and MUST not exceed the operative receive limit.

The checksum is CRC-32C using the Castagnoli polynomial `0x1EDC6F41`, reflected polynomial `0x82F63B78`, initial value `0xFFFFFFFF`, reflected input and output, and final XOR `0xFFFFFFFF` (LINK-FRM-039). The check value for the ASCII bytes `123456789` is `0xE3069283`. The CRC input is exactly `envelope_version || frame_length || generic_frame`. The transmitted CRC is unsigned big-endian. Delimiters and COBS-encoded bytes are not CRC input. CRC-32C detects corruption; it does not authenticate a sender or record.

### 5.2 COBS encoding

The profile uses standard Consistent Overhead Byte Stuffing (COBS), not COBS/R or another variant (LINK-FRM-040). An encoded candidate contains no zero byte. Each non-zero code byte denotes a block containing the following `code - 1` data bytes; a decoder inserts one zero after that block when the code is not `0xFF` and another encoded block follows. A code that extends beyond the candidate is invalid. Senders MUST produce the shortest standard encoding. Receivers MUST reject a candidate whose decode followed by canonical re-encoding differs from the received candidate.

For a maximum generic frame, decoded envelope content is at most 65,542 bytes and its canonical COBS encoding is at most 65,801 bytes (LINK-FRM-041). Including two delimiters, a standalone maximum record is at most 65,803 bytes.

### 5.3 Parser and recovery

At stream attachment, reset, or loss of synchronisation, the parser MUST discard input until it observes `0x00` (LINK-FRM-042). That delimiter establishes the start boundary for the next candidate.

Consecutive zero delimiters contain no candidate and MUST be treated as harmless idle separators (LINK-FRM-043).

For each non-empty candidate, the parser MUST keep total processing linear in the bytes received and MUST enforce the encoded and decoded bounds while receiving and decoding it (LINK-FRM-044). Bounded passes over decoded storage and local incremental canonicality checks are permitted, but repeated or super-linear processing of received stream bytes is not. The parser then validates, in order, canonical COBS encoding, envelope version, exact decoded length, declared frame length, CRC-32C, and the generic frame. A frame MUST NOT cross upward until every applicable check passes.

On malformed COBS, an unsupported envelope version, a length mismatch, an oversized candidate, a failed CRC, or an invalid generic frame, the parser MUST discard only the current candidate and resume from its terminating delimiter (LINK-FRM-045). As soon as safe incremental decoding has produced the three-byte envelope prefix, an unsupported envelope version or a declared frame length outside 3 through the operative receive limit MUST enter discard-until-delimiter recovery. If the encoded or decoded bound is exceeded before a delimiter arrives, decoding MUST stop and recovery MUST discard through the next delimiter. A malformed prefix alone MUST NOT make the link unavailable.

Every binding using this profile with a defensible positive progress guarantee MUST declare a maximum supported encoded-record length, a positive minimum encoded-byte delivery rate, a maximum aggregate permitted pause per record, and a finite scheduling and processing allowance (LINK-FRM-046). These values and the derived `maximum_record_lifetime` MUST be exposed to a conformance harness. Using exact arithmetic with division rounded up to the implementation's time unit, the lifetime MUST equal:

```text
maximum_record_lifetime =
    ceiling(maximum_supported_encoded_record_length / minimum_encoded_byte_rate)
    + maximum_aggregate_permitted_pause
    + scheduling_and_processing_allowance
```

The maximum encoded-record length MUST NOT exceed 65,801 bytes. The declared pause and allowance MUST each be finite. Delivery that exceeds the declared aggregate pause is not within the profile's progress guarantee.

A binding that cannot defend a positive minimum encoded-byte rate MUST instead define a finite profile-specific progress rule and deterministic abandonment or binding-reset outcome (LINK-FRM-047). The rule MUST bound how long and how much state any incomplete candidate may retain, MUST be exposed to a conformance harness, and MUST support the profile's declared intermittent operation. This requirement does not add capability descriptor fields or impose one duration on dissimilar media.

When `maximum_record_lifetime` expires, the parser MUST release the candidate's state, enter discard-until-delimiter recovery, and classify the candidate as `stream-record-timeout` without changing link availability solely for that timeout (LINK-FRM-048).

Parser storage MUST remain within 65,801 encoded bytes and 65,542 decoded bytes per active canonical stream record, and total parser work MUST remain linear in bytes consumed, including any final validation passes (LINK-FRM-049). An implementation MAY use less storage or incremental decoding.

### 5.4 Transmission commitment

For this profile, a generic frame becomes irreversibly committed when the first non-zero byte of its COBS-encoded record is accepted by the underlying stream (LINK-FRM-050). A leading zero delimiter alone does not commit a frame because it contains no frame-specific material.

Before commitment, the shim MAY cancel or return the exact frame as permitted by the section 02 barrier rules. After commitment, it MUST NOT return the frame as uncommitted because a frame-specific prefix may be observable by the peer (LINK-FRM-051). Completion and failure reporting for committed transmission are defined in section 06.

### 5.5 Established native adapter profiles

Every medium mapping MUST use a peer-shared adapter profile, including a direct one-record mapping. A profile other than the canonical bare-stream profile MUST provide all of the following (LINK-FRM-052):

- exact complete generic-frame boundaries;
- a total generic-frame bound no greater than 65,535 bytes;
- finite buffer, work, progress or dwell, and lifetime bounds;
- deterministic resynchronisation or a precise binding-reset condition and lifecycle result;
- CRC-32C covering the exact reconstructed generic frame, or a documented and tested whole-frame mechanism whose error-detection properties are at least as strong as CRC-32C for every supported frame length under the profile's declared error model;
- a precise irreversible transmission commit boundary;
- byte-identical generic-frame reconstruction.

For a non-CRC-32C mechanism, the profile MUST state its algorithm, exact covered bytes, supported lengths, declared error model, and evidence demonstrating that its undetected-corruption properties are no weaker than CRC-32C for those lengths and that model (LINK-FRM-053). Fragment-local checks do not satisfy this requirement unless the documented composition supplies the required whole-frame guarantee.

No native profile may expose partial generic frames, carry adaptation state across a retired binding or availability epoch, or use a malformed native record to change availability except through its explicitly defined reset condition (LINK-FRM-054).

Native framing integrity does not authenticate a generic frame. A profile MUST preserve any stronger native security property it advertises and MUST NOT claim that corruption detection supplies authenticity, confidentiality, replay protection, or end-to-end integrity (LINK-FRM-055).

## 6. Language-neutral conformance vectors

Hexadecimal octets in this section are separated by spaces. These vectors are normative examples of the framing rules (LINK-FRM-056).

### 6.1 Valid generic frames

Minimal Data frame with one body byte `AB`:

```text
01 01 03 AB
```

Data frame with body `AA BB` and four explicit padding bytes:

```text
01 01 07 FF 02 00 04 AA BB 00 00 00 00
```

Its derived values are `header_length = 7`, `padding_length = 4`, and `body_length = 2`.

### 6.2 Invalid generic frames

Each row is invalid independently:

| Bytes or condition | Reason |
|---|---|
| `01 01 02` | header length below 3 |
| `01 01 04` | header length exceeds complete frame length |
| `00 01 03 AA` | invalid frame version |
| `01 00 03 AA` | invalid frame class |
| `01 01 05 02 02 AA` | TLV value extends beyond the declared header |
| `01 02 0B FF 02 00 01 01 02 00 01 00` | recognised TLVs are not strictly increasing |
| `01 01 07 FF 02 00 00` | explicit zero padding length |
| `01 01 07 FF 02 00 04 AA` | padding length exceeds bytes after header |
| `01 01 07 FF 02 00 01 AA 01` | explicit padding byte is non-zero |
| `01 01 03` | empty Data body |
| `01 02 03` | Control Type absent |
| complete length one byte above the operative receive limit | `receive-frame-size-exceeded` |

### 6.3 Canonical stream record

For generic frame `01 01 03 AB`, the CRC input and result are:

```text
CRC input: 01 00 04 01 01 03 AB
CRC-32C:   44 59 76 D7
```

The decoded envelope content is:

```text
01 00 04 01 01 03 AB 44 59 76 D7
```

Its canonical COBS encoding is:

```text
02 01 0A 04 01 01 03 AB 44 59 76 D7
```

The complete standalone stream record is:

```text
00 02 01 0A 04 01 01 03 AB 44 59 76 D7 00
```

Changing any covered byte without recomputing the CRC produces a failed-CRC discard. Removing the final delimiter leaves an incomplete candidate, which is abandoned on `maximum_record_lifetime` expiry and recovery continues at the next delimiter.

### 6.4 Required boundary and recovery cases

A conformance suite MUST exercise at least the following cases and verify the stated discard classification and parser end-state where applicable (LINK-FRM-057):

- complete frame sizes of 255, 256, 65,535, and 65,536 bytes against limits for which each value is relevant;
- an operative directional limit exactly met and exceeded by one byte;
- TLV value lengths of 0 and 255, including one permitted and one forbidden zero-length definition when such a field is allocated;
- standard COBS blocks with code `0xFF`, the exact decoded and encoded maxima, and one byte beyond each maximum;
- every possible input chunk boundary for the section 6.3 record, including one byte per chunk;
- a corrupt candidate followed by a valid candidate, proving recovery at the delimiter;
- receipt immediately before and at lifetime or progress expiry;
- a partial record fenced by availability-epoch termination and prohibited from contributing to the next epoch;
- a size reduction with frames on both sides of the transmission commit boundary.

An invalid case MUST identify one expected primary validation failure where practical. When earlier validation intentionally masks a later defect, the conformance expectation MUST name the earlier failure.

## 7. Security and implementation freedom

Frame lengths, timing, class, header fields, and padding length are visible to the medium. The generic frame supplies no confidentiality, sender authentication, replay protection, or end-to-end integrity. CRC-32C and native checks detect accidental record corruption only.

An implementation may choose its internal buffer representation, allocation strategy, incremental parser structure, scheduling, and diagnostic storage. It may process in place or copy. Those choices are conforming only when all wire bytes, validation outcomes, resource bounds, discard isolation, ordering boundaries, and observable classifications defined here remain unchanged.

## 8. Requirement summary

This summary is a navigation aid. The normative prose above is authoritative.

- **LINK-FRM-001**: Every medium uses the same generic frame at the shim boundary, reconstructed byte-identically after native adaptation.
- **LINK-FRM-002**: Only complete generic frames cross the shim boundary; their complete length is supplied by that boundary or its adapter profile.
- **LINK-FRM-003**: A version 1 generic frame consists of a three-byte fixed header, ordered TLVs, a body, and optional explicit Link padding.
- **LINK-FRM-004**: Version 1 is `0x01`; version zero and unsupported versions are discarded before variable-header or body interpretation.
- **LINK-FRM-005**: Frame class `0x01` is Data and `0x02` is Control; zero and unknown classes are discarded before variable-header or body interpretation.
- **LINK-FRM-006**: Header length includes fixed and variable headers and is from 3 through 255 without exceeding complete frame length.
- **LINK-FRM-007**: A Data frame carries one complete Network packet or one fragment and has a non-empty body.
- **LINK-FRM-008**: A Control frame carries Link-owned protocol information and may have an empty body only when its Control Type permits it.
- **LINK-FRM-009**: A TLV is one-byte Type, one-byte Length, and exactly Length value bytes; zero length requires explicit permission.
- **LINK-FRM-010**: TLV types are strictly increasing; duplicates, reordering, truncation, and overrun are malformed.
- **LINK-FRM-011**: Every TLV must be recognised, allowed for its class, and valid in length and value; otherwise the frame is malformed.
- **LINK-FRM-012**: Version 1 assigns `0x00` invalid, `0x01` Control Type, `0x02` to `0xFE` to later Link sections, and `0xFF` Padding Length.
- **LINK-FRM-013**: Every allocated version 1 TLV type has exactly one meaning across the Link Specification.
- **LINK-FRM-014**: Control Type occurs exactly once in Control frames and never in Data frames; it is an unsigned 16-bit big-endian identifier.
- **LINK-FRM-015**: Every allocated version 1 Control Type has exactly one meaning across the Link Specification, and unallocated values are not transmitted.
- **LINK-FRM-016**: Padding Length is an optional two-byte final-padding count; absence means zero and an explicit zero is malformed.
- **LINK-FRM-017**: Senders encode explicit Link padding as zero bytes; receivers reject non-zero padding and remove valid padding; native and Transport padding remain separate.
- **LINK-FRM-018**: Body length is complete frame length minus header length minus padding length.
- **LINK-FRM-019**: Later sections use this format's fields and cannot create implementation-specific body boundaries.
- **LINK-FRM-020**: The guaranteed minimum MTU is 256 bytes.
- **LINK-FRM-021**: The global maximum generic frame size is 65,535 bytes.
- **LINK-FRM-022**: Every operative descriptor frame-size value is within 256 through 65,535 inclusive.
- **LINK-FRM-023**: Send and receive limits are direction-specific as defined in section 3.2.
- **LINK-FRM-024**: Maximum body capacity is operative send MTU minus actual header length and explicit padding length.
- **LINK-FRM-025**: Maximum packet size may exceed MTU and is defined by section 04 using framing, fragmentation, and reassembly bounds.
- **LINK-FRM-026**: The Link service validates structure and send MTU before `Shim-Send`; an invalid frame never reaches native transmission.
- **LINK-FRM-027**: The shim rejects intrinsic directional-size or adapter-profile violations as `rejected-malformed` and transient bounded-capacity exhaustion as `rejected-queue-full`; full generic-frame validation remains service-owned.
- **LINK-FRM-028**: The shim and Link service independently enforce the operative receive limit before frame parsing.
- **LINK-FRM-029**: Oversize reception is observably discarded as `receive-frame-size-exceeded` without affecting availability or unrelated state.
- **LINK-FRM-030**: A completed generic frame is validated in the order defined in section 4.1.
- **LINK-FRM-031**: Other validation failures are isolated `malformed-frame` discards.
- **LINK-FRM-032**: No field or body semantic is acted upon before its structural access bounds pass.
- **LINK-FRM-033**: Unvalidated lengths cannot control allocation, copying, iteration, or pointer arithmetic.
- **LINK-FRM-034**: A bare stream without a safe established profile uses the canonical stream profile.
- **LINK-FRM-035**: A canonical record is zero delimiter, COBS-encoded version, length, frame, and CRC-32C, then zero delimiter.
- **LINK-FRM-036**: Adjacent canonical records may share a delimiter.
- **LINK-FRM-037**: Canonical envelope version is `0x01`; zero and unsupported values are invalid.
- **LINK-FRM-038**: Frame length is an unsigned big-endian 16-bit count of generic-frame bytes and must match exactly.
- **LINK-FRM-039**: CRC-32C uses the stated Castagnoli parameters over version, frame length, and generic frame and is transmitted big-endian.
- **LINK-FRM-040**: The profile uses canonical shortest standard COBS and rejects non-canonical or malformed candidates.
- **LINK-FRM-041**: Decoded content is bounded at 65,542 bytes and encoded content at 65,801 bytes.
- **LINK-FRM-042**: Attachment, reset, and loss of synchronisation discard through a zero delimiter.
- **LINK-FRM-043**: Consecutive delimiters are harmless idle separators.
- **LINK-FRM-044**: Candidate parsing has total linear work, bounded storage, and completes all envelope checks before upward delivery.
- **LINK-FRM-045**: An impossible prefix is rejected as soon as safely known; a malformed or oversized candidate is discarded through its delimiter and does not itself make the link unavailable.
- **LINK-FRM-046**: A canonical stream binding with a positive progress guarantee declares its record length, minimum rate, aggregate pause, and processing allowance and derives its exposed lifetime by the specified equation.
- **LINK-FRM-047**: A binding without a defensible positive minimum rate uses an exposed finite progress rule and deterministic abandonment or reset that bounds incomplete-candidate state and supports its declared intermittent operation.
- **LINK-FRM-048**: Lifetime expiry releases candidate state and enters delimiter recovery as `stream-record-timeout` without itself changing availability.
- **LINK-FRM-049**: Per-record parser storage and work obey the exact byte bounds and linear-work rule.
- **LINK-FRM-050**: Canonical stream transmission commits at the first non-zero encoded-record byte accepted by the underlying stream.
- **LINK-FRM-051**: A committed frame cannot be returned as uncommitted; section 02 barrier rules govern either side of commitment.
- **LINK-FRM-052**: Every medium mapping uses a peer-shared adapter profile; a non-canonical profile supplies every listed boundary, safety, recovery, integrity, commitment, and reconstruction property.
- **LINK-FRM-053**: A non-CRC-32C adapter integrity mechanism documents its algorithm, exact coverage, lengths, error model, and evidence of whole-frame error detection no weaker than CRC-32C.
- **LINK-FRM-054**: Native profiles cannot expose partial frames, carry retired-episode state forward, or change availability for malformed input outside a defined reset condition.
- **LINK-FRM-055**: Native framing integrity is not authentication and cannot be represented as stronger security properties.
- **LINK-FRM-056**: Section 6 contains normative language-neutral examples of the framing rules.
- **LINK-FRM-057**: A conformance suite exercises the required size, TLV, COBS, chunking, recovery, expiry, epoch, and commitment boundary cases with expected classifications and parser states.
