# Status Message Payloads Specification

> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Abstract

This specifications decribes how the payload of each message in Status looks
like. It is primarly centered around chat and chat-related use cases.

The payloads aims be flexible enough to support messaging but also cases
described in the [Status Whitepaper](https://status.im/whitepaper.pdf) as well
as various clients created using different technologies.

## Table of Contents

- [Status Message Payloads Specification](#status-message-payloads-specification)
  - [Abstract](#abstract)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Payload wrapper](#payload-wrapper)
  - [Encoding](#encoding)
  - [Types of Messages] (#types-of-messages)
    - [Message](#message)
      - [Payload](#payload)
      - [Content types](#content-types)
      - [Message types](#message-types)
      - [Clock vs Timestamp and message ordering](#clock-vs-timestamp-and-message-ordering)
      - [Chats](#chats)
    - [Contact requests](#contact-requests)
      - [Payload] (#payload)
      - [Contact update] (#contact-update)
      - [Handling contact messages] (#handling-contact-messages)
    - [SyncInstallation](#sync-installation)
      - [Payload](#payload)
    - [PairInstallation](#pair-installation)
      - [Payload](#payload)
    - [GroupMembershipUpdate](#group-membership-update)
      - [Payload](#payload)
  - [Upgradability](#upgradability)
  - [Security Considerations](#security-considerations)
  - [Design rationale](#design-rationale)
    - [Why are you using Transit and Protobuf?](#why-are-you-using-transit-and-protobuf)

## Introduction

In this document we describe the payload format and some special considerations.

## Payload wrapper

All payloads are wrapped in a [protobuf record](https://developers.google.com/protocol-buffers/)
record:

```protobuf
message StatusProtocolMessage {
  bytes signature = 4001;
  bytes payload = 4002;
}
```

`signature` is the bytes of the signed `SHA3-256` of the payload, signed with the key of the author of the message.
The signature is needed to validate authorship of the message, so that the message can be relayed to third parties.
If a signature is not present but an author is provided by a layer below, the message is not to be relayed to third parties and its considered plausibly deniable.

## Encoding

The payload is encoded using [Transit format](https://github.com/cognitect/transit-format). Transit was chosen over JSON in order to reduce the bandwidth.

## Types of messages

### Message

The type `Message` represents a text message exchanged between clients and is identified by the transit tag `c4`.

#### Payload

Payload is a struct (a compound data type) with the following fields (order is important):

<!-- TODO: Be more precise in struct description, a la RFC, e.g. TLS style https://tools.ietf.org/html/rfc8446 -->

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | text | `string` | The text version of the message content |
| 2 | content type | `enum` (more in [Content types](#content-types)) | See details |
| 3 | message type | `enum` (more in [Message types](#message-types)) | See details |
| 4 | clock | `int64` | See details |
| 5 | timestamp | `int64` | See details |
| 6 | content | `struct { chat-id string, text string, response-to string }` | The chat-id of the chat this message is destined to, the text of the content and optionally the id of the message it is responding to|

#### Content types

Content types are required for a proper interpretation of incoming messages. Not each message is a plain text but may carry a different information.

The following content types MUST be supported:
* `text/plain` identifies a message which content is a plain text.

There are also other content types that MAY be implemented by the client:
* `sticker`
* `status`
* `command`
* `command-request`
* `emoji`

These are currently underspecified. We refer to real-world implementations for clients who wish to interoperate.

<!-- TODO: Ideally specify this, but barring that, link to implementation. -->

#### Message types

Message types are required to decide how a particular message is encrypted and what metadata needs to be attached when passing a message to the transport layer. For more on this, see [Status Whisper Usage Specification](./status-whisper-usage-spec.md).

<!-- TODO: This reference is a bit odd, considering the layer payloads should interact with is Secure Transport, and not Whisper. This requires more detail -->


The following messages types MUST be supported:
* `public-group-user-message` is a message to the public group
* `user-message` is a private message
* `group-user-message` is a message to the private group.

#### Clock vs Timestamp and message ordering

`timestamp` MUST be Unix time calculated when the message is created in milliseconds. This field SHOULD not be relied upon for message ordering.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, `clock`'s value is calculated based on the last received message in a particular chat: `last-message-clock-value + 1`. If there are no messages, `clock` is initialized with `timestamp * 100`'s value.

`clock` value is used for the message ordering. Due to the used algorithm and distributed nature of the system, we achieve casual ordering which might produce counterintuitive results in some edge cases. For example, when one joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

#### Chats

Chat is a structure that helps organize messages. It's usually desired to display messages only from a single recipient or a group of recipients at a time and chats help to achieve that.

All incoming messages can be matched against a chat. Below you can find a table that describes how to calculate a chat ID for each message type.

|Message Type|Chat ID Calculation|Direction|Comment|
|------------|-------------------|---------|-------|
|public-group-user-message|chat ID is equal to a public channel name; it should equal `chat-id` from message's `content` field|Incoming/Outgoing||
|user-message|let `P` be a public key of the recipient; `hex-encode(P)` is a chat ID; use it as `chat-id` value in message's `content` field|Outgoing||
|user-message|let `P` be a public key of message's signature; `hex-encode(P)` is a chat ID; discard `chat-id` from message's `content` field|Incoming|if there is no matched chat, it might be the first message from public key `P`; you can discard it or create a new chat; Status official clients create a new chat|
|group-user-message|use `chat-id` from message's `content` field|Incoming/Outgoing|find an existing chat by `chat-id`; if none is found discard the message (TODO: incomplete)|

<!-- TODO: "group-user-message" is not complete. Does it require to explicitly join the group chat? Is there a way to invite someone? Also, if I start a new group chat (or join an existing one), I need to somehow calculate this chatID by myself. How to do it? -->

### Contact Requests

These messages are used to notify the receiving end that it has been added to the sender's contact. They are identified by the transit tags `c2`, `c3`, `c4` respectively, but they are all interchangeable, meaning a client SHOULD handle them in exactly the same way. 

#### Payload

Payload is a struct (a compound data type) with the following fields (order is important):


| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | name | `string` | The self-assigned name of the user (DEPRECATED) |
| 2 | profile image | `string` | The base64 encoded profile picture of the user |
| 3 | address | `string` | The ethereum address of the user <!-- Why do we need this? can it be computed from the pk? --> |
| 4 | fcm-token | `string` | The FCM Token used by mobile devices for push notifications (DEPRECATED) |
| 5 | device-info | `[struct { id string, fcm-token string }]` | A list of pair `installation-id`, `fcm-token` for each device that is currently paired |

#### Contact update

A client SHOULD send a `ContactUpdate` to all the contacts each time:

- The name is edited
- The profile image is edited
- A new device has been paired

A client SHOULD also periodically send a `ContactUpdate` to all the contacts, the interval is up to the client.


#### Handling contact messages

A client SHOULD handle any `Contact*` message in the same way. Any `Contact*` message with a whisper timestamp lower than the last one processed MUST be discarded.

### SyncInstallation

`SyncInstallation` messages are used to synchronize in a best-effort way all the paired installations. It is identified by a transit tag of `p1` 

#### Payload

Payload is a struct (a compound data type) with the following fields (order is important):


| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1| contacts | `[struct { name string last-updated int device-info struct {id string fcm-token string } pending? bool}` | An array of contacts |
| 2 | account | `struct {name string photo-path string last-updated int}` | Information about your own account |
| 3 | chat | `struct {:public? bool :chat-id string}` | A description of a public chat opened by the client |

### PairInstallation

`PairInstallation` messages are used to propagate informations about a device to its paired devices. It is identified by a transit tag of `p2` 

#### Payload

Payload is a struct (a compound data type) with the following fields (order is important):


| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1| installation-id | `string` | A randomly generated id that identifies this device |
| 2 | device-type | `string` | The OS of the device `ios`,`android` or `desktop` |
| 3 | name | `string` | The self-assigned name of the device |
| 4 | fcm-token | `string` | The FCM Token used by mobile devices for push notifications |

### GroupMembershipUpdate

`GroupMembershipUpdate` is a message used to propagate information about group membership changes in a group chat.. It is identified by a transit tag of `g5`.
The details are in the  [Group chats specs](status-group-chats-spec.md)

#### Payload

Payload is a struct (a compound data type) with the following fields (order is important):


| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1| chat-id | `string` | The chat id of the chat where the change is to take place |
| 2 | membership-updates | See details | A list of events that describe the membership changes |
| 3 | message | `Transit message` | An optional message, described in [Message](#message) |

## Upgradability

There are two ways to upgrade the protocol without breaking compatibility:

- Map fields can be enriched with a new key, which will be ignored by old clients.
- An element can be appended to the `Transit` array, which will also be ignored by old clients.

## Security Considerations

TBD.

## Design rationale

### Why are you using Transit and Protobuf?

Transit was initially chose for encoding, and Protobuf was added afterwards. This is partly due to the history of the protocol living inside of `status-react`, which is written in Clojurescript.
