# Specifications for Status clients

This repository contains a list of specifications for implementing Status and
its various capabilities.

## Current state

As of August 2019, we are currently in the process of documenting current
specifications. We are als implementing an isolated reference library for them.
These specifications are expected to be frozen at the Status V1 launch and be
used as a reference point for client implementers and security audits.

## Status Improvement Proposals (SIPs)

### Accepted

No accepted SIPs right now.

### Draft

The following SIPs are under consideration for standardization.

- [Status Client Specification](status-spec.md)
- [Initial Conversational Security Specification](status-secure-transport-spec.md)
- [Initial Message Payload Specification](status-payloads-spec.md)
- [Status Account Specification](status-account-spec.md)
- [Status Whisper Usage Specification](status-whisper-usage-spec.md)

Badly scoped:

- [Initial Trust Establishment Specification](x5.md)

## Protocol Research

These are protocols that are currently being researched. These are designed to
be useful outside of Status as well. To the extent that these protocols are used
within Status clients, they will show up as SIPs in the future.

To see more on this, please visit the current home: [vac
protocol](https://specs.vac.dev).
