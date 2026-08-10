# Link Specification

## 1. Purpose

This specification defines the OpenNet Link layer: the contract a link exposes to the Network layer above it, the contract a shim MUST satisfy to adapt a medium into a compliant link, and the mechanisms by which opaque payloads are carried over heterogeneous media.

The Link layer MUST be specified independently of any upper layer (LINK-OVR-001). The Link layer MUST treat every payload it carries as an opaque byte sequence and MUST NOT inspect, parse, or act on payload contents (LINK-OVR-002).

A medium is adapted into a compliant link by a shim that satisfies the shim interface defined in section 02 and reports a capability descriptor describing the medium's properties (LINK-OVR-019). The native API of a medium is out of scope: this specification defines what sits on the OpenNet side of the boundary, not what a medium must expose (LINK-OVR-020).

## 2. Scope

### 2.1 In scope

- The upward service contract exposed to the Network Specification.
- The shim interface a medium adapter MUST satisfy, and the capability descriptor a shim MUST report.
- Media enumeration, observation, and link selection policy.
- Service primitives, link lifecycle, and multi-link multiplexing.
- Adaptation for arbitrary physical and link carriers, including Ethernet, Wi-Fi, Bluetooth, LoRa, arbitrary datagram links, stream links, packet-oriented links, high-latency links, lossy links, intermittent links, one-way links, broadcast and multicast media, audio, optical, and unconventional links. These are non-normative examples; this specification does not enumerate media.
- Framing, maximum transmission unit (MTU), and maximum packet size.
- Fragmentation and reassembly within the Link layer.
- Link-level addressing and neighbour discovery.
- Link quality, loss, retransmission, ordering, and duplication.
- Link-local congestion and maximum packet size interactions.
- Energy and constraint considerations for constrained devices.

The detailed requirements for each in-scope area are owned by the sections listed in section 5; this overview defines cross-cutting invariants only.

### 2.2 Out of scope

- Network-layer addressing and packet format. Defined in the Network Specification.
- Session and transport semantics. Defined in the Transport Specification.
- Cryptographic details. Defined in the Transport Specification.

## 3. Design objectives

The Link layer is designed to operate over essentially any medium capable of carrying a signal, from constrained devices to powerful computers. Five objectives govern this specification.

Objective 1: Medium independence. The Link layer MUST NOT assume the properties of any specific medium (LINK-OVR-003). In particular, a compliant link MUST NOT assume support for a particular frame format, a particular addressing scheme, a particular delivery semantic, a particular reliability level, or a particular maximum frame size unless that support is reported by the shim through the capability descriptor defined in section 02.

Objective 2: Opacity. The Link layer carries opaque payloads. It MUST NOT require knowledge of network-layer structure, transport-layer structure, or cryptographic structure to perform its functions (LINK-OVR-004).

Objective 3: Implementability. The mechanisms defined here MUST be implementable without contacting any central authority, registry, allocator, certificate authority, or governance body (LINK-OVR-005).

Objective 4: Extensibility of media. New media MUST be supportable by writing a new shim, without revising this specification (LINK-OVR-021). An implementation MAY ship shims for a finite subset of media; additional shims are implementation work and do not affect v1 compliance.

Objective 5: Generic frame. The Link layer defines a single generic frame format used across all media (LINK-OVR-022). The generic frame is the complete unit exchanged between the link service and a shim. A shim MUST translate between that unit and the medium's native transport on send and receive, preserving the generic frame exactly. A translation MAY segment, aggregate, delimit, or reassemble native transport units below the shim boundary when the medium cannot carry one generic frame as one native unit. Such adaptation MUST expose only complete generic frames to the link service, MUST NOT expose a partial generic frame, and is subject to the resource and hostile-input requirements of LINK-OVR-024. The frame format itself is defined normatively in section 03; section 02 defines the adaptation contract and when a shared adaptation profile is required.

## 4. Conformance

An implementation conforms to this specification if and only if it satisfies every implementation requirement identified by a `LINK-*` requirement ID within this specification.

Requirements in this specification fall into two classes. Implementation requirements bind a running implementation and are testable against it; conformance of an implementation is judged solely against implementation requirements. Specification-suite invariants bind the authors of this specification and of the other specifications in the OpenNet suite; they are not implementation conformance targets. Both classes share the same requirement-ID space. Each section's requirement summary identifies which of its requirements are specification-suite invariants.

Where a requirement is conditional on a property reported by a shim's capability descriptor (for example, a reliability property or a maximum frame size), conformance applies when and only when that property is present. Conformance to media-specific behaviour that is reported as absent is not required (LINK-OVR-006).

An implementation MAY ship a subset of shims for any media. An implementation that ships only one shim conforms if it satisfies the requirements applicable to that medium and to the link service contract (LINK-OVR-007).

Requirement IDs provide stable handles between this specification and its conformance artefacts, including language-agnostic test vectors and automated conformance tests.

## 5. Structure

This specification is composed of the following sections. Each section carries its own normative requirements; this overview establishes framing and cross-cutting invariants only. Where a later section appears to conflict with a cross-cutting invariant established in this overview, the invariant prevails; the later section is in error.

| Section | Title |
|---------|-------|
| 01 | Link Service Model |
| 02 | Medium Adaptation Shims |
| 03 | Framing and MTU |
| 04 | Fragmentation and Reassembly |
| 05 | Link Addressing and Neighbour Discovery |
| 06 | Link Quality, Loss, and Retransmission |
| 07 | Ordering and Duplication |
| 08 | Energy and Constrained Devices |

## 6. Layering

The Link layer is the bottom layer of the OpenNet stack. The Network Specification builds on the service contract defined in section 01 and MUST NOT require link behaviour outside this specification (LINK-OVR-008).

The Link layer carries opaque payloads and depends on no upper layer. Where this specification references behaviour defined elsewhere, the reference is informative; normative authority for that behaviour lives in the referenced specification.

## 7. Terminology

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in RFC 2119.

Terms defined in the project glossary apply. The following terms are used throughout this specification and are defined here for the Link layer.

**link**
A unit of connectivity exposed upward by this specification. A link is produced by a shim adapting a medium and presenting the service contract defined in section 01.

**medium**
A physical or logical carrier capable of transporting a signal between two or more points. A medium has a native API owned by the medium, not by this specification. A medium is adapted into a link by a shim.

**shim**
The implementation that bridges a medium and the link service. A shim satisfies the shim interface defined in section 02, reports a capability descriptor describing the medium's properties, and translates between the medium's native transport and the link service contract. The translation may use bounded medium-side segmentation, aggregation, delimiting, or reassembly while exposing only complete generic frames at the shim boundary.

**Capability descriptor**
A structured report of a medium's properties produced by a shim. Its reported properties include maximum frame size, reliability, ordering, broadcast support, multicast support, full-duplex support, intermittence, one-way direction, latency class, throughput class, and energy class. This list is illustrative, not exhaustive; the normative definition is in section 02.

**Payload**
A byte sequence carried by a link. Payloads are opaque to the Link layer.

**Frame**
The complete unit exchanged between the link service and a shim, defined by this specification as a single generic frame format used across all media (LINK-OVR-022). A frame carries one or more payload fragments, control information, and link-layer addressing as defined in sections 03 and 05. A shim translates between the generic frame and the medium's native transport; one generic frame need not correspond to exactly one native transport unit.

**Maximum transmission unit (MTU)**
The maximum frame size a link reports it can carry without further fragmentation. The MTU varies per link between a guaranteed minimum and a global maximum frame size; both bounds are defined in section 03.

**Maximum packet size**
The maximum payload size the Link layer accepts from the Network layer. Defined in section 03.

**Node**
A device or entity that implements the OpenNet stack and participates in communication over one or more links.

**Neighbour**
A node reachable over a single link without an intermediate forwarder.

## 8. Byte order, encodings, and serialisation

All multi-byte integer fields defined by this specification MUST be in unsigned big-endian order unless stated otherwise in the field definition (LINK-OVR-009).

All lengths, counts, and sizes defined by this specification MUST be in bytes unless stated otherwise in the field definition (LINK-OVR-010).

Canonical serialisation of any structure defined by this specification MUST be unambiguous: for any given structure, all conforming transmitters MUST produce the same byte sequence, and a receiver MUST reconstruct field values identical to those transmitted. Duplicate fields in a single structure MUST NOT be permitted (LINK-OVR-011). Treatment of reserved or unused bits is defined normatively where the containing fields are defined.

Where this specification accepts a structure from an upper layer for opaque transport, the Link layer does not define its serialisation. The defining specification governs serialisation of that structure (LINK-OVR-012).

## 9. Malformed input and limits

A receiver of a link-layer frame MUST define behaviour for malformed input, including truncation, length-field mismatch, and field values outside the specified range (LINK-OVR-013). Such frames MUST be discarded. The discard of a malformed frame MUST NOT affect frames, flows, or link-layer state unrelated to that frame, and MUST NOT in itself cause the link to become unavailable (LINK-OVR-014). Link availability transitions are governed by the mechanisms defined in section 06.

This specification defines limits for fields, structures, and counts where they are needed for interoperation. Where a limit is not stated, the limit is bounded by the reported MTU of the link (LINK-OVR-015).

The MTU of a link is variable per link. Every compliant link MUST support at least the guaranteed minimum MTU and MUST NOT exceed the global maximum frame size; both values are defined in section 03 (LINK-OVR-029). Limits bounded by the reported MTU (LINK-OVR-015) are further bounded by the global maximum frame size.

Where this specification states a maximum, an implementation MUST NOT generate values exceeding the maximum (LINK-OVR-016). Where this specification states a minimum, an implementation MUST NOT generate values below the minimum (LINK-OVR-017).

## 10. Security considerations

The Link layer is the first layer of the OpenNet stack to process input from a medium, and the medium is assumed hostile. Frames may be malformed, spoofed, replayed, flooded, or crafted to exhaust the resources of a receiver. The invariants in this section govern every later section; the sections that define the corresponding mechanisms define the concrete bounds.

The Link layer provides no authenticity, integrity, or confidentiality services. All input received from a medium MUST be treated as unauthenticated and adversarial (LINK-OVR-023). Cryptographic protection is defined in the Transport Specification (see section 2.2).

All link-layer state, including shim-private segmentation and reassembly state, stream-parser state, link-layer reassembly contexts, neighbour state, and retransmission buffers, MUST be resource-bounded, and that state MUST remain correct under adversarial input (LINK-OVR-024). Concrete byte, context, work, and lifetime bounds are defined by the sections that define the corresponding state.

Link-layer addressing MUST NOT embed or expose upper-layer identity or long-term device identity (LINK-OVR-025). Neighbour discovery MUST tolerate unauthenticated adversarial input: spoofed, flooded, or replayed discovery traffic MUST NOT corrupt state belonging to other nodes or flows (LINK-OVR-026). Section 05 defines the addressing and discovery mechanisms that satisfy these invariants.

The capability descriptor reported by a shim is host-local input, trusted within the host trust boundary. Before a descriptor value is used to size state or select behaviour, an implementation MUST validate it against the bounds and consistency rules defined in section 02 (LINK-OVR-027). Where safety-relevant behaviour is conditional on a shim-reported property and that property is absent or invalid, the condition MUST fail closed: the implementation MUST behave as if the property were absent (LINK-OVR-028).

The Link layer leaks metadata by design: frame sizes, timing, and link-layer addressing are visible to every node on the medium. Opacity (LINK-OVR-002) is a conformance invariant binding conforming implementations, not an enforcement mechanism; it cannot prevent a non-conforming shim from inspecting payloads.

## 11. Reserved requirement-ID prefix

All normative requirements in this specification carry a stable identifier in the form `LINK-AREA-NNN`, where `AREA` is one of:

- `OVR`: overview and cross-cutting invariants (this section)
- `SVC`: link service model (section 01)
- `MED`: medium adaptation shims (section 02)
- `FRM`: framing and MTU (section 03)
- `FRG`: fragmentation and reassembly (section 04)
- `ADR`: link addressing and neighbour discovery (section 05)
- `QRT`: link quality, loss, and retransmission (section 06)
- `ORD`: ordering and duplication (section 07)
- `ENG`: energy and constrained devices (section 08)

Requirement IDs in this specification MUST NOT collide with requirement IDs in any other specification (LINK-OVR-018).

## 12. Requirement summary

The following normative requirements are defined in this overview section. Subsequent sections define their own. This summary restates the tagged requirements in the body for reference; the tagged statements in the body are authoritative. Restatements of requirement content elsewhere in this section (for example, the in-scope list in section 2.1) carry no additional requirements. Entries marked (specification-suite invariant) bind the specification suite rather than a running implementation; see section 4.

- **LINK-OVR-001** (specification-suite invariant): The Link layer MUST be specified independently of any upper layer.
- **LINK-OVR-002**: The Link layer MUST treat every payload as an opaque byte sequence and MUST NOT inspect, parse, or act on payload contents.
- **LINK-OVR-003**: The Link layer MUST NOT assume the properties of any specific medium, including frame format, addressing scheme, delivery semantic, reliability level, or maximum frame size, unless that support is reported by the shim through the capability descriptor defined in section 02.
- **LINK-OVR-004**: The Link layer MUST NOT require knowledge of network-layer structure, transport-layer structure, or cryptographic structure to perform its functions.
- **LINK-OVR-005**: The mechanisms defined in this specification MUST be implementable without contacting any central authority, registry, allocator, certificate authority, or governance body.
- **LINK-OVR-006**: Where a requirement is conditional on a property reported by a shim's capability descriptor, conformance applies when and only when that property is present.
- **LINK-OVR-007**: An implementation MAY conform by shipping a subset of shims, provided it satisfies the requirements applicable to that medium and to the link service contract.
- **LINK-OVR-008** (specification-suite invariant): The Network Specification MUST NOT require link behaviour outside this specification.
- **LINK-OVR-009**: All multi-byte integer fields defined by this specification MUST be in unsigned big-endian order unless stated otherwise in the field definition.
- **LINK-OVR-010**: All lengths, counts, and sizes defined by this specification MUST be in bytes unless stated otherwise in the field definition.
- **LINK-OVR-011**: Canonical serialisation of any structure defined by this specification MUST be unambiguous. For any given structure, all conforming transmitters MUST produce the same byte sequence, and a receiver MUST reconstruct field values identical to those transmitted. Duplicate fields in a single structure MUST NOT be permitted.
- **LINK-OVR-012** (specification-suite invariant): Where this specification accepts a structure from an upper layer for opaque transport, the Link layer does not define its serialisation. The defining specification governs serialisation of that structure.
- **LINK-OVR-013**: A receiver of a link-layer frame MUST define behaviour for malformed input, including truncation, length-field mismatch, and field values outside the specified range.
- **LINK-OVR-014**: Malformed frames MUST be discarded. The discard of a malformed frame MUST NOT affect frames, flows, or link-layer state unrelated to that frame, and MUST NOT in itself cause the link to become unavailable.
- **LINK-OVR-015**: Where this specification does not state a limit, the limit is bounded by the reported MTU of the link.
- **LINK-OVR-016**: Where this specification states a maximum, an implementation MUST NOT generate values exceeding the maximum.
- **LINK-OVR-017**: Where this specification states a minimum, an implementation MUST NOT generate values below the minimum.
- **LINK-OVR-018** (specification-suite invariant): Requirement IDs in this specification MUST NOT collide with requirement IDs in any other specification.
- **LINK-OVR-019**: A medium is adapted into a compliant link by a shim that satisfies the shim interface defined in section 02 and reports a capability descriptor describing the medium's properties.
- **LINK-OVR-020** (specification-suite invariant): The native API of a medium is out of scope. This specification defines what sits on the OpenNet side of the boundary, not what a medium must expose.
- **LINK-OVR-021**: New media MUST be supportable by writing a new shim, without revising this specification. An implementation MAY ship shims for a finite subset of media; additional shims are implementation work and do not affect v1 compliance.
- **LINK-OVR-022**: The Link layer defines a single generic frame format used across all media. The generic frame is the complete unit exchanged between the link service and a shim. A shim MUST translate between that unit and the medium's native transport on send and receive, preserving the generic frame exactly. The translation MAY segment, aggregate, delimit, or reassemble native transport units below the shim boundary, but MUST expose only complete generic frames to the link service and MUST NOT expose a partial generic frame.
- **LINK-OVR-023**: The Link layer provides no authenticity, integrity, or confidentiality services. All input received from a medium MUST be treated as unauthenticated and adversarial.
- **LINK-OVR-024**: All link-layer state, including shim-private segmentation and reassembly state, stream-parser state, link-layer reassembly contexts, neighbour state, and retransmission buffers, MUST be resource-bounded, and that state MUST remain correct under adversarial input. Concrete byte, context, work, and lifetime bounds are defined by the sections that define the corresponding state.
- **LINK-OVR-025**: Link-layer addressing MUST NOT embed or expose upper-layer identity or long-term device identity.
- **LINK-OVR-026**: Neighbour discovery MUST tolerate unauthenticated adversarial input. Spoofed, flooded, or replayed discovery traffic MUST NOT corrupt state belonging to other nodes or flows.
- **LINK-OVR-027**: The capability descriptor is host-local input, trusted within the host trust boundary. Before a descriptor value is used to size state or select behaviour, an implementation MUST validate it against the bounds and consistency rules defined in section 02.
- **LINK-OVR-028**: Where safety-relevant behaviour is conditional on a shim-reported property and that property is absent or invalid, the condition MUST fail closed: the implementation MUST behave as if the property were absent.
- **LINK-OVR-029**: Every compliant link MUST support at least the guaranteed minimum MTU and MUST NOT exceed the global maximum frame size; both values are defined in section 03. Limits bounded by the reported MTU (LINK-OVR-015) are further bounded by the global maximum frame size.
