---
permalink: /spec/6
parent: Draft specs
title: 6/PAYLOADS
---

# 6/PAYLOADS

> Version: 0.3
>
> Status: Draft
>
> Authors: Adam Babik <adam@status.im>, Andrea Maria Piana <andreap@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Abstract

This specification describes how the payload of each message in Status looks
like. It is primarily centered around chat and chat-related use cases.

The payloads aims to be flexible enough to support messaging but also cases
described in the [Status Whitepaper](https://status.im/whitepaper.pdf) as well
as various clients created using different technologies.

## Table of Contents

 - [Abstract](#abstract)
 - [Table of Contents](#table-of-contents)
 - [Introduction](#introduction)
 - [Payload wrapper](#payload-wrapper)
 - [Encoding](#encoding)
 - [Types of messages](#types-of-messages)
   - [Message](#message)
     - [Payload](#payload)
     - [Payload](#payload-1)
     - [Content types](#content-types)
       - [Sticker content type](#sticker-content-type)
       - [Image content type](#image-content-type)
       - [EmojiReaction content type](#emojireaction-content-type)
       - [EmojiReactionRetraction content type](#emojireactionretraction-content-type)
     - [Message types](#message-types)
     - [Clock vs Timestamp and message ordering](#clock-vs-timestamp-and-message-ordering)
     - [Chats](#chats)
   - [Contact Update](#contact-update)
     - [Payload](#payload-2)
     - [Contact update](#contact-update-1)
   - [SyncInstallationContact](#syncinstallationcontact)
     - [Payload](#payload-3)
   - [SyncInstallationPublicChat](#syncinstallationpublicchat)
     - [Payload](#payload-4)
   - [PairInstallation](#pairinstallation)
     - [Payload](#payload-5)
   - [MembershipUpdateMessage and MembershipUpdateEvent](#membershipupdatemessage-and-membershipupdateevent)
 - [Upgradability](#upgradability)
 - [Security Considerations](#security-considerations)

## Introduction

This document describes the payload format and some special considerations.

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
The node needs the signature to validate authorship of the message, so that the message can be relayed to third parties.
If a signature is not present, but an author is provided by a layer below, the message is not to be relayed to third parties, and it is considered plausibly deniable.

## Encoding

The node encodes the payload using [Protobuf](https://developers.google.com/protocol-buffers)

## Types of messages

### Message

The type `ChatMessage` represents a chat message exchanged between clients.

#### Payload

The protobuf description is:

```protobuf
message ChatMessage {
  // Lamport timestamp of the chat message
  uint64 clock = 1;
  // Unix timestamps in milliseconds, currently not used as we use whisper as more reliable, but here
  // so that we don't rely on it
  uint64 timestamp = 2;
  // Text of the message
  string text = 3;
  // Id of the message that we are replying to
  string response_to = 4;
  // Ens name of the sender
  string ens_name = 5;
  // Chat id is the ID of the chat
  string chat_id = 6;

  // The type of message (public/one-to-one/private-group-chat)
  MessageType message_type = 7;
  // The type of the content of the message
  ContentType content_type = 8;

  oneof payload {
    StickerMessage sticker = 9;
    ImageMessage image = 10;
    EmojiReaction emoji_reaction = 11;
    EmojiReactionRetraction emoji_reaction_retraction = 12;
  }

  enum MessageType {
    UNKNOWN_MESSAGE_TYPE = 0;
    ONE_TO_ONE = 1;
    PUBLIC_GROUP = 2;
    PRIVATE_GROUP = 3;
    // Only local
    SYSTEM_MESSAGE_PRIVATE_GROUP = 4;}
  enum ContentType {
    UNKNOWN_CONTENT_TYPE = 0;
    TEXT_PLAIN = 1;
    STICKER = 2;
    STATUS = 3;
    EMOJI = 4;
    TRANSACTION_COMMAND = 5;
    // Only local
    SYSTEM_MESSAGE_CONTENT_PRIVATE_GROUP = 6;
    IMAGE = 7;
    EMOJI_REACTION = 8;
    EMOJI_REACTION_RETRACTION = 9;
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
| 9 | payload | `Sticker` I `Image` I `EmojiReaction` I `EmojiReactionRetraction` I `nil` | The payload of the message based on the content type |

#### Content types

A node requires content types for a proper interpretation of incoming messages. Not each message is plain text but may carry different information.

The following content types MUST be supported:
* `TEXT_PLAIN` identifies a message which content is a plaintext.

There are other content types that MAY be implemented by the client:
* `STICKER`
* `STATUS`
* `EMOJI`
* `TRANSACTION_COMMAND`
* `IMAGE`
* `EMOJI_REACTION`
* `EMOJI_REACTION_RETRACTION`

##### Sticker content type

A `ChatMessage` with `STICKER` `Content/Type` MUST also specify the ID of the `Pack` and 
the `Hash` of the pack, in the `Sticker` field of `ChatMessage`

```protobuf
message StickerMessage {
  string hash = 1;
  int32 pack = 2;
}
```

##### Image content type

A `ChatMessage` with `IMAGE` `Content/Type` MUST also specify the `payload` of the image
and the `type`

```protobuf
message ImageMessage {
  bytes payload = 1;
  ImageType type = 2;
  enum ImageType {
    UNKNOWN_IMAGE_TYPE = 0;
    PNG = 1;
    JPEG = 2;
    WEBP = 3;
    GIF = 4;
  }
}
```

##### EmojiReaction content type

`EmojiReaction`s represents a user's "reaction" to a specific chat message. For more information about the concept of
emoji reactions see [Facebook Reactions](https://en.wikipedia.org/wiki/Facebook_like_button#Use_on_Facebook).

This specification RECOMMENDS that the UI/UX implementation of sending `EmojiReactions` requires only a single click
operation, as users have an expectation that emoji reactions are effortless and simple to perform.  

A `ChatMessage` with `EMOJI_REACTION` `Content/Type` MUST also specify the `EmojiReaction` fields `message_id` and `type`

```protobuf
message EmojiReaction {
  string message_id = 1;
  Type type = 2;

  enum Type {
    UNKNOWN_EMOJI_REACTION_TYPE = 0;
    LOVE = 1;
    THUMBS_UP = 2;
    THUMBS_DOWN = 3;
    LAUGH = 4;
    SAD = 5;
    ANGRY = 6;
  }
}
```

#####  EmojiReactionRetraction content type

`EmojiReactionRetraction`s represent a user removing a reaction, they previously performed, from a specific chat message.

This specification RECOMMENDS that the UI/UX implementation of sending `EmojiReactionRetraction`s requires only a single
click operation, as users have an expectation that emoji reaction removals are effortless and simple to perform.  

A `ChatMessage` with `EMOJI_REACTION_RETRACTION` `Content/Type` MUST also specify the `EmojiReactionRetraction` field
`emoji_reaction_id`

```protobuf
message EmojiReactionRetraction {
  string emoji_reaction_id = 1;
}
```

#### Message types

A node requires message types to decide how to encrypt a particular message and what metadata needs to be attached when passing a message to the transport layer. For more on this, see [3/WHISPER-USAGE](./../stable/3-whisper-usage.md) and [10/WAKU-USAGE](./../stable/10-waku-usage.md).

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

`timestamp` MUST be Unix time calculated, when the node creates the message, in milliseconds. This field SHOULD not be relied upon for message ordering.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, the node calculates `clock`'s value based on the last received message in a particular chat: `max(timeNowInMs, last-message-clock-value + 1)`. If there are no messages, `clock` is initialized with `timestamp`'s value.

Messages with a `clock` greater than `120` seconds over the whisper timestamp SHOULD be discarded, in order to avoid malicious users to increase the `clock` of a chat arbitrarily.

Messages with a `clock` less than `120` seconds under the whisper timestamp might indicate an attempt to insert messages in the chat history which is not distinguishable from a `datasync` layer re-transit event. A client MAY mark this messages with a warning to the user, or discard them.

The node uses `clock` value for the message ordering. The algorithm used, and the distributed nature of the system produces casual ordering, which might produce counter-intuitive results in some edge cases. For example, when a user joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

#### Chats

Chat is a structure that helps organize messages. It's usually desired to display messages only from a single recipient, or a group of recipients at a time and chats help to achieve that.

All incoming messages can be matched against a chat. The below table describes how to calculate a chat ID for each message type.

|Message Type|Chat ID Calculation|Direction|Comment|
|------------|-------------------|---------|-------|
|PUBLIC_GROUP|chat ID is equal to a public channel name; it should equal `chatId` from the message|Incoming/Outgoing||
|ONE_TO_ONE|let `P` be a public key of the recipient; `hex-encode(P)` is a chat ID; use it as `chatId` value in the message|Outgoing||
|user-message|let `P` be a public key of message's signature; `hex-encode(P)` is a chat ID; discard `chat-id` from message|Incoming|if there is no matched chat, it might be the first message from public key `P`; the node MAY discard the message or MAY create a new chat; Status official clients create a new chat|
|PRIVATE_GROUP|use `chatId` from the message|Incoming/Outgoing|find an existing chat by `chatId`; if none is found, the user is not a member of that chat or the user hasn't joined that chat, the message MUST be discarded |

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
- A user edits the profile image

A client SHOULD also periodically send a `ContactUpdate` to all the contacts, the interval is up to the client, the Status official client sends these updates every 48 hours.

### SyncInstallationContact

The node uses `SyncInstallationContact` messages to synchronize in a best-effort the contacts to other devices.

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

The node uses `SyncInstallationPublicChat` message to synchronize in a best-effort the public chats to other devices.

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

The node uses `PairInstallation` messages to propagate information about a device to its paired devices.

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
The details are in the [Group chats specs](./7-group-chat.md).

## Upgradability

There are two ways to upgrade the protocol without breaking compatibility:

- A node always supports accretion
- A node does not support deletion of existing fields or messages, which might break compatibility

## Security Considerations

-

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
