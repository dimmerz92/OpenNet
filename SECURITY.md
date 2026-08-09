# Security policy

## Reporting a vulnerability

Do not open a public issue for an undisclosed exploitable vulnerability.

Use GitHub's private vulnerability reporting feature for this repository. Include:

- affected specification sections or implementation components;
- affected requirement IDs, where known;
- attack prerequisites and threat model;
- reproduction details or a proof of concept;
- impact and likely affected deployments;
- suggested mitigations, if available;
- whether the issue concerns the specification, implementation, or both.

If private vulnerability reporting is unavailable, do not publish exploit details.
Open a minimal public issue asking the repository owner to establish a private
contact channel.

## Scope

Security reports may concern protocol design, cryptography, downgrade or replay,
resource exhaustion, malformed inputs, privacy leakage, implementation safety,
dependency vulnerabilities, or build and release integrity.

## Disclosure

Please allow time to reproduce, assess, and remedy a report before public
disclosure. A specification vulnerability found before v1 may reopen affected
sections. Published v1 remains immutable. Post-v1 findings may lead to errata,
implementation mitigations, advisories, extensions, or successor work without
silently changing v1.
