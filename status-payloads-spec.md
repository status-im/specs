# Status Message Payloads Specification

> Version: 0.2
>
> Status: Stable
>
> Authors: Adam Babik <adam@status.im>, Andrea Maria Piana <andreap@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

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
    - [ContactUpdate](#contact-update)
      - [Payload] (#payload)
    - [SyncInstallationContact](#sync-installation-contact)
      - [Payload](#payload)
    - [SyncInstallationPublicChat](#sync-installation-public-chat)
      - [Payload](#payload)
    - [PairInstallation](#pair-installation)
      - [Payload](#payload)
    - [MembershipUpdateMessage](#membership-update-message)
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

The payload is encoded using [Protobuf](https://developers.google.com/protocol-buffers)

## Types of messages

### Message

The type `ChatMessage` represents a chat message exchanged between clients.

#### Payload

The protobuf description is:

```protobuf
message ChatMessage {
  // Lamport timestamp of the chat message
  uint64 clock = 1;
  // Unix timestamp in milliseconds, not 
  uint64 timestamp = 2;
  // Text of the message
  string text = 3;
  // Id of the message that we are replying to
  string response_to = 4;
  // Ens name of the sender
  string ens_name = 5;
  // Local chatID of the message
  string chat_id = 6;

  // The type of message (public/one-to-one/private-group-chat)
  MessageType message_type = 7;
  // The type of the content of the message
  ContentType content_type = 8;

  oneof payload {
    StickerMessage sticker = 9;
  }

  enum MessageType {
    UNKNOWN_MESSAGE_TYPE = 0;
    ONE_TO_ONE = 1;
    PUBLIC_GROUP = 2;
    PRIVATE_GROUP = 3;
    // Only local
    SYSTEM_MESSAGE_PRIVATE_GROUP = 4;
    }
  enum ContentType {
    UNKNOWN_CONTENT_TYPE = 0;
    TEXT_PLAIN = 1;
    STICKER = 2;
    STATUS = 3;
    EMOJI = 4;
    TRANSACTION_COMMAND = 5;
  }
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | The clock of the chat|
| 2 | timestamp | `uint64` | The sender timestamp at message creation |
| 3 | text | `string` | The content of the message |
| 4 | response_to | `string` | The ID of the message replied to |
| 5 | ens_name | `string` | The ENS name of the user sending the message |
| 6 | chat_id | `string` | The local ID of the chat the message is sent to |
| 7 | message_type | `MessageType` | The type of message, different for one-to-one, public or group chats |
| 8 | content_type | `ContentType` | The type of the content of the message | 
| 9 | payload | `Sticker|nil` | The payload of the message based on the content type |

#### Content types

Content types are required for a proper interpretation of incoming messages. Not each message is plain text but may carry a different information.

The following content types MUST be supported:
* `TEXT_PLAIN` identifies a message which content is a plaintext.

There are also other content types that MAY be implemented by the client:
* `STICKER`
* `STATUS`
* `EMOJI`
* `TRANSACTION_COMMAND`

##### Sticker content type

A `ChatMessage` with `STICKER` `Content/Type` MUST also specify the ID of the `Pack` and 
the `Hash` of the pack, in the `Sticker` field of `ChatMessage`

```protobuf
message StickerMessage {
  string hash = 1;
  int32 pack = 2;
}
```

#### Message types

Message types are required to decide how a particular message is encrypted and what metadata needs to be attached when passing a message to the transport layer. For more on this, see [Status Whisper Usage Specification](./status-whisper-usage-spec.md).

<!-- TODO: This reference is a bit odd, considering the layer payloads should interact with is Secure Transport, and not Whisper. This requires more detail -->


The following messages types MUST be supported:
* `ONE_TO_ONE` is a message to the public group
* `PUBLIC_GROUP` is a private message
* `PRIVATE_GROUP` is a message to the private group.

#### Clock vs Timestamp and message ordering

If a user sends a new message before the messages sent while the user was offline are received, the new
message is supposed to be displayed last in a chat. This is where the basic algorithm of Lamport timestamp would fall short
as it's only meant to order causally related events.

The status client therefore makes a "bid", speculating that it will beat the current chat-timestamp, s.t. the status client's
Lamport timestamp format is: `clock = `max({timestamp}, chat_clock + 1)`

This will satisfy the Lamport requirement, namely: a -> b then T(a) < T(b)

`timestamp` MUST be Unix time calculated when the message is created in milliseconds. This field SHOULD not be relied upon for message ordering.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, `clock`'s value is calculated based on the last received message in a particular chat: `max(timeNowInMs, last-message-clock-value + 1)`. If there are no messages, `clock` is initialized with `timestamp`'s value.

Messages with a `clock` greater than `120` seconds over the whisper timestamp SHOULD be discarded, in order to avoid malicious users to increase the `clock` of a chat arbitrarily.

Messages with a `clock` less than `120` seconds under the whisper timestamp might indicate an attempt to insert messages in the chat history which is not distinguishable from a `datasync` layer re-transit event. A client MAY mark this messages with a warning to the user, or discard them.

`clock` value is used for the message ordering. Due to the used algorithm and distributed nature of the system, we achieve casual ordering which might produce counterintuitive results in some edge cases. For example, when one joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

#### Chats

Chat is a structure that helps organize messages. It's usually desired to display messages only from a single recipient or a group of recipients at a time and chats help to achieve that.

All incoming messages can be matched against a chat. Below you can find a table that describes how to calculate a chat ID for each message type.

|Message Type|Chat ID Calculation|Direction|Comment|
|------------|-------------------|---------|-------|
|PUBLIC_GROUP|chat ID is equal to a public channel name; it should equal `chatId` from the message|Incoming/Outgoing||
|ONE_TO_ONE|let `P` be a public key of the recipient; `hex-encode(P)` is a chat ID; use it as `chatId` value in the message|Outgoing||
|user-message|let `P` be a public key of message's signature; `hex-encode(P)` is a chat ID; discard `chat-id` from message|Incoming|if there is no matched chat, it might be the first message from public key `P`; you can discard it or create a new chat; Status official clients create a new chat|
|PRIVATE_GROUP|use `chatId` from the message|Incoming/Outgoing|find an existing chat by `chatId`; if none is found, we are not a member of that chat or we haven't joined that chat, the message MUST be discarded |

### Contact Update

`ContactUpdate` is a message exchange to notify peers that either the
user has been added as a contact, or that information about the sending user have
changed.

```protobuf
message ContactUpdate {
  uint64 clock = 1;
  string ens_name = 2;
  string profile_image = 3;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | The clock of the chat with the user |
| 2 | ens_name | `string` | The ENS name if set |
| 3 | profile_image | `string` | The base64 encoded profile picture of the user |

#### Contact update

A client SHOULD send a `ContactUpdate` to all the contacts each time:

- The ens_name has changed
- The profile image is edited

A client SHOULD also periodically send a `ContactUpdate` to all the contacts, the interval is up to the client, the Status official client sends these updates every 48 hours.

### SyncInstallationContact

`SyncInstallationContact` messages are used to synchronize in a best-effort the contacts to other devices.

```protobuf
message SyncInstallationContact {
  uint64 clock = 1;
  string id = 2;
  string profile_image = 3;
  string ens_name = 4;
  uint64 last_updated = 5;
  repeated string system_tags = 6;
}
```


#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | clock value of the chat | 
| 2 | id | `string` | id of the contact synced |
| 3 | profile_image | `string` |  `base64` encoded profile picture of the user |
| 4 | ens_name | `string` | ENS name of the contact |
| 5 | `array[string]` | Array of `system_tags` for the user, this can currently be: `":contact/added", ":contact/blocked", ":contact/request-received"`|

### SyncInstallationPublicChat

`SyncInstallationPublicChat` message is used to synchronize in a best-effort the public chats to other devices.

```protobuf
message SyncInstallationPublicChat {
  uint64 clock = 1;
  string id = 2;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | clock value of the chat | 
| 2 | id | `string` | id of the chat synced |

### PairInstallation

`PairInstallation` messages are used to propagate informations about a device to its paired devices.

```protobuf
message PairInstallation {
  uint64 clock = 1;
  string installation_id = 2;
  string device_type = 3;
  string name = 4;
}
```

#### Payload

| Field | Name | Type | Description |
| ----- | ---- | ---- | ---- |
| 1 | clock | `uint64` | clock value of the chat | 
| 2| installation_id | `string` | A randomly generated id that identifies this device |
| 3 | device_type | `string` | The OS of the device `ios`,`android` or `desktop` |
| 4 | name | `string` | The self-assigned name of the device |

### MembershipUpdateMessage and MembershipUpdateEvent

`MembershipUpdateEvent` is a message used to propagate information about group membership changes in a group chat.
The details are in the  [Group chats specs](status-group-chats-spec.md)

## Upgradability

There are two ways to upgrade the protocol without breaking compatibility:

- Accretion is always supported
- Deletion of existing fields or messages is not supported and might break compatibility

## Security Considerations

-

## Design rationale
