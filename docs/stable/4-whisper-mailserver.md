---
permalink: /spec/4
parent: Stable specs
title: 4/WHISPER-MAILSERVER
---

# 4/WHISPER-MAILSERVER

> Version: 0.3
>
> Status: Stable
>
> Authors: Adam Babik <adam@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

 - [Abstract](#abstract)
 - [`Mailserver`](#mailserver)
   - [Archiving messages](#archiving-messages)
   - [Requesting messages](#requesting-messages)
   - [Receiving historic messages](#receiving-historic-messages)
 - [Security considerations](#security-considerations)
   - [Confidentiality](#confidentiality)
   - [Altruistic and centralized operator risk](#altruistic-and-centralized-operator-risk)
   - [Privacy concerns](#privacy-concerns)
   - [Denial-of-service](#denial-of-service)
 - [Changelog](#changelog)
   - [Version 0.3](#version-03)

## Abstract

Being mostly offline is an intrinsic property of mobile clients. They need to save network transfer and battery consumption to avoid spending too much money or constant charging. Whisper protocol, on the other hand, is an online protocol. Messages are available in the Whisper network only for short period of time calculate in seconds.

Whisper `Mailserver` is a Whisper extension that allows to store messages permanently and deliver them to the clients even though they are already not available in the network and expired.

## `Mailserver`

From the network perspective, `Mailserver` is just like any other Whisper node. The only difference is that it has a capability of archiving messages and delivering them to its peers on-demand.

It is important to notice that `Mailserver` will only handle requests from its direct peers and exchanged packets between `Mailserver` and a peer are p2p messages.

### Archiving messages

A node which wants to provide `Mailserver` functionality MUST store envelopes from incoming message packets (Whisper packet-code `0x01`). The envelopes can be stored in any format, however they MUST be serialized and deserialized to the Whisper envelope format.

A `Mailserver` SHOULD store envelopes for all topics to be generally useful for any peer, however for specific use cases it MAY store envelopes for a subset of topics.

### Requesting messages

In order to request historic messages, a node MUST send a packet P2P Request (`0x7e`) to a peer providing `Mailserver` functionality. This packet requires one argument which MUST be a Whisper envelope.

In the Whisper envelope's payload section, there MUST be RLP-encoded information about the details of the request:

```
[ Lower, Upper, Bloom, Limit, Cursor ]
```

`Lower`: 4-byte wide unsigned integer (UNIX time in seconds; oldest requested envelope's creation time)  
`Upper`: 4-byte wide unsigned integer (UNIX time in seconds; newest requested envelope's creation time)  
`Bloom`: 64-byte wide array of Whisper topics encoded in a bloom filter to filter envelopes  
`Limit`: 4-byte wide unsigned integer limiting the number of returned envelopes  
`Cursor`: an array of a cursor returned from the previous request (optional)

The `Cursor` field SHOULD be filled in if a number of envelopes between `Lower` and `Upper` is greater than `Limit` so that the requester can send another request using the obtained `Cursor` value. What exactly is in the `Cursor` is up to the implementation. The requester SHOULD NOT use a `Cursor` obtained from one `Mailserver` in a request to another `Mailserver` because the format or the result MAY be different.

The envelope MUST be encrypted with a symmetric key agreed between the requester and `Mailserver`.

### Receiving historic messages

Historic messages MUST be sent to a peer as a packet with a P2P Message code (`0x7f`) followed by an array of Whisper envelopes. It is incompatible with the original Whisper spec (EIP-627) because it allows only a single envelope, however, an array of envelopes is much more performant. In order to stay compatible with EIP-627, a peer receiving historic message MUST handle both cases.

In order to receive historic messages from a `Mailserver`, a node MUST trust the selected `Mailserver`, that is allowed to receive packets with the P2P Message code. By default, the node discards such packets.

Received envelopes MUST be passed through the Whisper envelope pipelines so that they are picked up by registered filters and passed to subscribers.

For a requester, to know that all messages have been sent by `Mailserver`, it SHOULD handle P2P Request Complete code (`0x7d`). This code is followed by the following parameters:

```
[ RequestID, LastEnvelopeHash, Cursor ]
```

`RequestID`: 32-byte wide array with a Keccak-256 hash of the envelope containing the original request  
`LastEnvelopeHash`: 32-byte wide array with a Keccak-256 hash of the last sent envelope for the request  
`Cursor`: an array of a cursor returned from the previous request (optional)

If `Cursor` is not empty, it means that not all messages were sent due to the set `Limit` in the request. One or more consecutive requests MAY be sent with `Cursor` field filled in order to receive the rest of the messages.

## Security considerations

### Confidentiality

The node encrypts all Whisper envelopes. A `Mailserver` node can not inspect their contents.

### Altruistic and centralized operator risk

In order to be useful, a `Mailserver` SHOULD be online most of the time. That means
users either have to be a bit tech-savvy to run their own node, or rely on someone
else to run it for them.

Currently, one of Status's legal entities provides `Mailservers` in an altruistic manner, but this is
suboptimal from a decentralization, continuance and risk point of view. Coming
up with a better system for this is ongoing research.

A Status client SHOULD allow the `Mailserver` selection to be customizable.

### Privacy concerns

In order to use a `Mailserver`, a given node needs to connect to it directly,
i.e. add the `Mailserver` as its peer and mark it as trusted. This means that the
`Mailserver` is able to send direct p2p messages to the node instead of
broadcasting them. Effectively, it will have access to the bloom filter of
topics that the user is interested in, when it is online as well as many
metadata like IP address.

### Denial-of-service

Since a `Mailserver` is delivering expired envelopes and has a direct TCP connection with the recipient, the recipient is vulnerable to DoS attacks from a malicious `Mailserver` node.

## Changelog

### Version 0.3

Released [May 22, 2020](https://github.com/status-im/specs/commit/664dd1c9df6ad409e4c007fefc8c8945b8d324e8)

- Change to keep `Mailserver` term consistent
