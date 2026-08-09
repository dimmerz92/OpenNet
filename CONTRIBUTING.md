# Contributing to OpenNet

OpenNet welcomes technically relevant contributions to the protocol, its
documentation, conformance material, and reference implementation.

## Keep discussion about the project

Contributions are assessed on technical merit, protocol correctness, security,
interoperability, implementation quality, evidence, and relevance. Technical
disagreement is welcome. Address the work, support claims with reasoning or
evidence, and leave personal attacks, harassment, inflammatory conduct, and
off-topic political, religious, or social advocacy elsewhere.

## Contribution terms

Original OpenNet material is made available under CC0 1.0 Universal. By submitting
a contribution, you affirm that you have the right to submit it and apply CC0 1.0
Universal to it to the fullest extent permitted by law. You understand that
attribution is not required and that the contribution may be used, modified,
redistributed, combined, or removed without credit or compensation.

Do not submit third-party material unless its licence permits inclusion and you
identify its source, version, licence, and applicable notice or source-code
obligations. Third-party and vendored material retains its own licence. Inclusion
in this repository does not relicense it under CC0.

Commit and hosting metadata may retain a factual record of participation even
though CC0 does not require attribution in copies or derived works.

## Development phases

### Before v1

Specifications remain open to controlled community refinement. A finalised
section has passed its development review, but may be reopened before v1 when
implementation, security, interoperability, or community evidence shows that a
revision is needed.

Specification changes must identify affected sections and requirement IDs, and
explain their interoperability, security, implementation, compatibility, and
conformance effects. An editorial change must not alter normative meaning.

### Publishing v1

V1 exists only after an explicit and intentional release. Ordinary merges, tags,
section finalisation, or implementation releases do not publish v1.

### After v1

The published v1 normative specification is immutable. Contributions may continue
to improve the reference implementation, tests, tooling, and non-normative
documentation, provided they remain compliant with v1.

Reports of possible v1 specification defects may be recorded as errata, but
cannot change what v1 requires. New normative behaviour belongs in a compatible
extension, profile, experimental successor, derivative, or fork, with its status
and compatibility stated clearly.

## Specification contributions before v1

- Identify the specification, section, and requirement IDs.
- State the current and proposed normative behaviour.
- Explain why the change is needed for interoperability, security, observable
  behaviour, resource safety, or objective conformance.
- Avoid mandating internal representations, APIs, algorithms, allocation,
  scheduling, or module structure unless necessary for those properties.
- Define byte order, encoding, malformed-input behaviour, failure behaviour, and
  bounds wherever they affect independent implementations.
- Include or propose language-neutral vectors when behaviour is objectively
  testable.

## Code contributions

- Implement only behaviour supported by the applicable specification.
- Cite relevant requirement IDs in code or associated traceability material.
- Include tests for behaviour changes and regressions.
- Cover malformed, negative, boundary, resource, and security cases where
  applicable.
- Do not guess when the specification is materially ambiguous. Before v1, open a
  specification ambiguity issue. After v1, open a conformance or erratum report.
- Preserve the implementation's documented portability and language requirements.

Post-v1 implementation changes are welcome when they remain compliant with the
immutable v1 specification.

## Test and vector contributions

Normative expected behaviour must come from the specification rather than a
particular implementation. Language-neutral vectors should be usable by
independent implementations. Report disagreement between a test and the
specification rather than silently choosing one.

## Documentation contributions

Documentation must describe the protocol accurately and must not introduce new
normative behaviour. If documentation and a specification disagree, report the
conflict explicitly.

## Tool-assisted contributions

Contributors remain responsible for every submitted change, including work made
with automated or generative tools. Review generated output, verify its licensing
provenance, remove unsupported claims, and meet the same security, correctness,
and testing standards as any other contribution.

## Security

Do not disclose an exploitable vulnerability in a public issue. Follow
[SECURITY.md](SECURITY.md) for private reporting.
