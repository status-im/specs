# Specifications for Status clients

This repository contains a list of specifications for implementing Status and
its various capabilities.

## Spec lifecycle

Every spec has its own lifecycle that shows its maturity. We indicate this in a similar fashion to [COSS Lifecycle](https://rfc.unprotocols.org/spec:2/COSS/):

![](assets/lifecycle.png)

## Status Improvement Proposals (SIPs)

### Stable

No stable specs right now. De facto a lot of the draft ones are stable and indicating this is work in progress.

### Draft

The following SIPs are under consideration for standardization.

- [Status Client Specification](status-client-spec.md). The main specification for writing a Status client. **Start here**
- [Status Secure Transport Specification](status-secure-transport-spec.md). How Status provide a secure transport with conversational security properties.
- [Status Payload Specification](status-payloads-spec.md). What the message payloads look like.
- [Status Account Specification](status-account-spec.md). What a Status account is and how trust is established.
- [Status Whisper Usage Specification](status-whisper-usage-spec.md). How we use Whisper to do routing, metadata protection and provide 1:1/group/public chat.
- [Status Whisper Mailserver Specification](status-whisper-mailserver-spec.md). How we use Whisper mailservers to provide offline inboxing.
- [Status EIPs Standards](status-EIPs.md). Ethereum Improvement Proposals used in Status.

### Raw

No raw specs right now.

### Deprecated

No deprecated specs right now.

## Protocol Research

These are protocols that are currently being researched. These are designed to
be useful outside of Status as well. To the extent that these protocols are used
within Status clients, they will show up as SIPs in the future.

To see more on this, please visit the current home: [vac
protocol](https://specs.vac.dev).
