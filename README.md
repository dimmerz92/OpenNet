# OpenNet

**You cannot stop the signal**

The advent of the internet was a masterpiece of order and chaos.
It gave us new ways to connect, share, create, and collaborate.

It has since calcified around centralised platforms, providers, authorities,
and infrastructure. What was once a frontier of freedom is now shaped by
surveillance, control, influence, and censorship.

OpenNet is a network stack designed to be unstoppable, uncensorable, and
ungovernable.

Anyone may participate without asking. There is no central authority, no
privileged infrastructure, and no master switch. The protocol grants nobody
authority over another.

OpenNet is a digital road. Nothing more. It is neither good nor bad. It simply
connects one place to another.

No tolls. No gatekeepers. Just the signal.

You cannot stop the signal.

## What OpenNet is

If two devices can exchange a signal, they should be able to form a network
without asking.

That signal might travel over Ethernet, Wi-Fi, Bluetooth, low-bandwidth radio,
audio, light, a serial connection, or something nobody has tried yet. New kinds
of links should be addable without redesigning the network above them.

OpenNet is divided into three layers:

```text
Transport   Secure sessions and end-to-end communication
Network     Addresses, packets, and forwarding between nodes
Link        Communication over a physical or logical medium
```

Each layer will have its own specification and may be implemented independently.
A complete reference implementation will join them into a stack that works out
of the box.

This is intended to become real, usable software, not merely a paper architecture
or thought experiment.

## Key features

The exact Network and Transport mechanisms are still being specified. OpenNet is
being designed to provide:

- **Post-quantum hybrid protection by default.** Protected communication is
  intended to combine classical and post-quantum cryptography rather than depend
  on either alone.
- **Forward secrecy.** Compromise of a long-term key should not expose previously
  protected sessions.
- **Self-certifying and ephemeral addresses.** Nodes should be able to create
  cryptographically meaningful addresses without a central allocator, including
  temporary addresses when persistent identity is unnecessary.
- **Plain and protected communication.** Applications should be able to use the
  network as a plain datagram substrate or use the secure Transport layer.
- **Privacy-conscious forwarding.** The network should reveal only what is needed
  to deliver traffic and avoid exposing higher-level meaning to intermediaries
  without necessity.

## Principles

- No authority should be required to join the network.
- No registry should be required to create an address or extend the system.
- No maintainer or organisation should be able to redefine the protocol for
  everyone else.
- No single transmission medium should be treated as the network itself.
- The network should carry information without unnecessarily understanding what
  it means.
- Privacy and anonymity should be preserved where possible, with their limits
  stated honestly.
- Independent implementations should be able to interoperate.
- The protocol should work on constrained devices as well as ordinary computers.
- Security claims should be precise and testable.

Decentralisation is not an optional feature. It is a constraint on the design
from the beginning.

## V1 and permanence

OpenNet v1 will be released explicitly and deliberately.

Before v1, the specifications remain open to implementation experience, security
review, and community refinement. Once v1 is published, its normative meaning
will be immutable. Neither the original author nor future maintainers will have
special power to rewrite it.

Implementations, compatible extensions, and successor protocols may continue to
develop. None may silently change v1 while claiming to remain compliant with it.

## Current status

OpenNet is in pre-v1 development and is not ready for deployment.

The Link specification, followed by the Network and Transport specifications,
must be completed before their corresponding reference implementations are built.

## Contributing

OpenNet welcomes technically relevant contributions from anyone. Arguments should
be settled by evidence, interoperability, security, and working implementations,
not status or affiliation.

See [CONTRIBUTING.md](CONTRIBUTING.md) before submitting work. Report exploitable
security issues according to [SECURITY.md](SECURITY.md), not in a public issue.

## No permission required

Original OpenNet material is dedicated to the public domain under
[CC0 1.0 Universal](LICENSE).

Use it. Implement it. Change it. Build upon it. No attribution is required.

The point is not to own the network. The point is to make a network that nobody
needs permission to use.
