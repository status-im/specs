---
sip: 3
title: Initial Message Payload Specification
status: Draft
type: Standard
author: Adam Babik <adam@status.im>, Oskar Thorén <oskar@status.im>
created: 2019-04-22
updated:
---

- [Wrapper](#wrapper)
- [Encoding](#encoding)
- [Upgradability](#upgradability)

This specification describes how the payload of each message in the Status Protocol looks.

The payload must be flexible enough to support messaging but also cases described in [Status Whitepaper](https://status.im/whitepaper.pdf) as well as various clients created using vastly different technologies.

# Wrapper

Payloads are wrapped in a [protobuf record](https://developers.google.com/protocol-buffers/)
record:

```protobuf
message StatusProtocolMessage {
  bytes signature = 1;
  bytes payload = 2;
}
```

`signature` is the bytes of the signed `SHA3-256` of the payload, signed with the key of the author of the message.
The signature is needed to validate authorship of the message, so that the message can be relayed to third parties.
If a signature is not present but an author is provided by a layer below, the message is to be relayed to third parties and its considered plausibly deniable.

# Encoding

The payload is encoded using [Transit format](https://github.com/cognitect/transit-format). Transit was chosen over JSON in order to reduce the bandwidth.

Example of a valid encoded payload:

```
["~#c4",["abc123","text/plain","~:public-group-user-message",154593077368201,1545930773682,["^ ","~:chat-id","testing-adamb","~:text","abc123"]]]
```

As you can see, the message is an array and each index value has its meaning:
* 0: `c4` is a decoder handler identification for the current payload format. Identifications allow to register handlers for many different types of payload
* 1: array which items correspond to the described payload fields above

For more details regarding serialization and deserialization please consult [transit format](https://github.com/cognitect/transit-format) specification.

<!--
# Message types

- [Message](#message)

-->

# Message

The type `Message` represents a text message exchanged between clients.

- [Payload](#payload)
- [Content types](#content-types)
- [Message types](#message-types)
- [Clock vs Timestamp and message ordering](#clock-vs-timestamp-and-message-ordering)
- [Replies](#replies)

## Payload

Payload is a struct (a compound data type) with the following fields (order is important):

<!-- TODO: Be more precise in struct description, a la RFC, e.g. TLS style https://tools.ietf.org/html/rfc8446 -->

| Field | Name | Type |
| ----- | ---- | ---- |
| 1 | text | `string` |
| 2 | content type | `enum` (more in [Content types](#content-types)) |
| 3 | message type | `enum` (more in [Message types](#message-types)) |
| 4 | clock | `int64` |
| 5 | timestamp | `int64` |
| 6 | content | `struct { chat-id string, text string }` |

## Content types

Content types are required for a proper interpretation of incoming messages. Not each message is a plain text but may carry a different information.

The following content types MUST be supported:
* `text/plain` identifies a message which content is a plain text.

There are also other content types that MAY be implemented by the client:
* `sticker` TODO
* `status` TODO
* `command` TODO
* `command-request` TODO
* `emoji` TODO

### Message types

Message types are required to decide how a particular message is encrypted (more in [Whisper > Message encryption](#message-encryption)) and what metadata needs to be attached (more in [Whisper > Topic](#topic)) when passing a message to the transport layer.

The following messages types MUST be supported:
* `public-group-user-message` is a message to the public group
* `user-message` is a private message
* `group-user-message` is a message to the private group.

## Clock vs Timestamp and message ordering

`timestamp` MUST be Unix time calculated when the message is created. Because the peers in the Whisper network should have synchronized time, `timestamp` values should be fairly accurate among all Whisper network participants.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, `clock`'s value is calculated based on the last received message in a particular chat: `last-message-clock-value + 1`. If there are no messages, `clock` is initialized with `timestamp`'s value.

`clock` value is used for the message ordering. Due to the used algorithm and distributed nature of the system, we achieve casual ordering which might produce counterintuitive results in some edge cases. For example, when one joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

## Replies

TODO

# Upgradability

The current protocol format is hardly upgradable without breaking backward compatibility. Because Transit is used in this particular way described above, the only reliable option is to append a new field to the Transit record definition. It will be simply ignored by the old clients.
