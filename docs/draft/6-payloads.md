---
permalink: /spec/6
parent: Draft specs
title: 6/PAYLOADS
---

# 6/PAYLOADS

> Version: 0.5
>
> Status: Draft
>
> Authors: Adam Babik <adam@status.im>, Andrea Maria Piana <andreap@status.im>, Oskar Thorén <oskar@status.im>, Samuel Hawksby-Robinson <samuel@status.im> (alphabetical order)

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
       - [Audio content type](#audio-content-type)
     - [Message types](#message-types)
     - [Clock vs Timestamp and message ordering](#clock-vs-timestamp-and-message-ordering)
     - [Chats](#chats)
   - [Chat Message Identity](#chat-message-identity)
   - [Contact Update](#contact-update)
     - [Payload](#payload-2)
     - [Contact update](#contact-update-1)
   - [EmojiReaction](#emojireaction)
   - [SyncInstallationContact](#syncinstallationcontact)
     - [Payload](#payload-3)
   - [SyncInstallationPublicChat](#syncinstallationpublicchat)
     - [Payload](#payload-4)
   - [PairInstallation](#pairinstallation)
     - [Payload](#payload-5)
   - [MembershipUpdateMessage and MembershipUpdateEvent](#membershipupdatemessage-and-membershipupdateevent)
 - [Upgradability](#upgradability)
 - [Security Considerations](#security-considerations)
 - [Changelog](#changelog)
 - [Copyright](#copyright)

## Introduction

This document describes the payload format and some special considerations.

## Payload wrapper

The node wraps all payloads in a [protobuf record](https://developers.google.com/protocol-buffers/)
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
  // Unix timestamps in milliseconds, currently not used as we use Whisper/Waku as more reliable, but here
  // so that we don't rely on it
  uint64 timestamp = 2;
  // Text of the message
  string text = 3;
  // Id of the message that we are replying to
  string response_to = 4;
  // DEPRECATED. Ens name of the sender
  string ens_name = 5;
  // Chat id, this field is symmetric for public-chats and private group chats,
  // but asymmetric in case of one-to-ones, as the sender will use the chat-id
  // of the received, while the receiver will use the chat-id of the sender.
  string chat_id = 6;

  // The type of message (public/one-to-one/private-group-chat)
  MessageType message_type = 7;
  // The type of the content of the message
  ContentType content_type = 8;

  oneof payload {
    StickerMessage sticker = 9;
    ImageMessage image = 10;
    AudioMessage audio = 11;
  }

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
    AUDIO = 8;
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
| 5 | ens_name | `string` | DEPRECATED - [See Chat Message Identity](#chat-message-identity). The ENS name of the user sending the message |
| 6 | chat_id | `string` | The local ID of the chat the message is sent to |
| 7 | message_type | `MessageType` | The type of message, different for one-to-one, public or group chats |
| 8 | content_type | `ContentType` | The type of the content of the message | 
| 9 | payload | `Sticker` I `Image` I `Audio` I `nil` | The payload of the message based on the content type |

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
* `AUDIO`

##### Mentions 

A mention MUST be represented as a string with the `@0xpk` format, where `pk` is the public key of the [user account](https://specs.status.im/spec/2) to be mentioned, within the `text` field of a message with content_type `TEXT_PLAIN`.
A message MAY contain more than one mention.
This specification RECOMMENDs that the application does not require the user to enter the entire pk. 
This specification RECOMMENDs that the application allows the user to create a mention by typing @ followed by the related ENS or 3-word pseudonym.
This specification RECOMMENDs that the application provides the user auto-completion functionality to create a mention.
For better user experience, the client SHOULD display a known [ens name or the 3-word pseudonym corresponding to the key](https://specs.status.im/spec/2#contact-verification) instead of the `pk`.

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
and the `type`.

Clients MUST sanitize the payload before accessing its content, in particular: 
- Clients MUST choose a secure decoder
- Clients SHOULD strip metadata if present without parsing/decoding it
- Clients SHOULD NOT add metadata/exif when sending an image file for privacy and security reasons
- Clients MUST make sure that the transport layer constraints the size of the payload to limit they are able to handle securely


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

##### Audio content type

A `ChatMessage` with `AUDIO` `Content/Type` MUST also specify the `payload` of the audio,
the `type` and the duration in milliseconds (`duration_ms`).

Clients MUST sanitize the payload before accessing its content, in particular: 
- Clients MUST choose a secure decoder
- Clients SHOULD strip metadata if present without parsing/decoding it
- Clients SHOULD NOT add metadata/exif when sending an audio file for privacy and security reasons
- Clients MUST make sure that the transport layer constraints the size of the payload to limit they are able to handle securely

```protobuf
message AudioMessage {
  bytes payload = 1;
  AudioType type = 2;
  uint64 duration_ms = 3;
  enum AudioType {
    UNKNOWN_AUDIO_TYPE = 0;
    AAC = 1;
    AMR = 2;
```

#### Message types

A node requires message types to decide how to encrypt a particular message and what metadata needs to be attached when passing a message to the transport layer. For more on this, see [3/WHISPER-USAGE](./../stable/3-whisper-usage.md) and [10/WAKU-USAGE](./../stable/10-waku-usage.md).

<!-- TODO: This reference is a bit odd, considering the layer payloads should interact with is Secure Transport, and not Whisper/Waku. This requires more detail -->


The following messages types MUST be supported:
* `ONE_TO_ONE` is a message to the public group
* `PUBLIC_GROUP` is a private message
* `PRIVATE_GROUP` is a message to the private group.

```protobuf
  enum MessageType {
    UNKNOWN_MESSAGE_TYPE = 0;
    ONE_TO_ONE = 1;
    PUBLIC_GROUP = 2;
    PRIVATE_GROUP = 3;
    // Only local
    SYSTEM_MESSAGE_PRIVATE_GROUP = 4;
}
```

#### Clock vs Timestamp and message ordering

If a user sends a new message before the messages sent while the user was offline are received, the new
message is supposed to be displayed last in a chat. This is where the basic algorithm of Lamport timestamp would fall short
as it's only meant to order causally related events.

The status client therefore makes a "bid", speculating that it will beat the current chat-timestamp, s.t. the status client's
Lamport timestamp format is: `clock = max({timestamp}, chat_clock + 1)`

This will satisfy the Lamport requirement, namely: a -> b then T(a) < T(b)

`timestamp` MUST be Unix time calculated, when the node creates the message, in milliseconds. This field SHOULD not be relied upon for message ordering.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, the node calculates `clock`'s value based on the last received message in a particular chat: `max(timeNowInMs, last-message-clock-value + 1)`. If there are no messages, `clock` is initialized with `timestamp`'s value.

Messages with a `clock` greater than `120` seconds over the Whisper/Waku timestamp SHOULD be discarded, in order to avoid malicious users to increase the `clock` of a chat arbitrarily.

Messages with a `clock` less than `120` seconds under the Whisper/Waku timestamp might indicate an attempt to insert messages in the chat history which is not distinguishable from a `datasync` layer re-transit event. A client MAY mark this messages with a warning to the user, or discard them.

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

### Chat Message Identity

The `ChatMessageIdentity` allows a user to OPTIONALLY broadcast an identity to be associated with their messages.

The main components of the `ChatMessageIdentity` are:

| Field name      | Description |
| --------------- |---|
| `ens_name`      | A valid registered ENS name for the user. Deprecates the `ens_name` field in `ChatMessage` |
| `display_name`  | A user determined display name not requiring blockchain registry |
| `profile_image` | A `ProfileImage` data struct used to transmit user profile image data |

#### Profile Image

The `ProfileImage` data struct describes the mechanisms by which the application parses and presents the user's chosen visual representation.

The main components of the `ProfileImage` are:

| Field name      | Description |
| --------------- |---|
| `payload` | A context based payload for the profile image data. Context is determined by the `source_type` |
| `source_type` | A `SourceType` enum, signals the image payload source |
| `image_type` | An `ImageType` enum, signals the image type and method of parsing the payload |

#### Payload

```protobuf
syntax="proto3";

// ChatMessageIdentity represents the user defined identity associated with their messages
message ChatMessageIdentity {
  // Lamport timestamp of the chat message
  uint64 clock = 1;

  // ens_name is the valid ENS name associated with the chat key
  string ens_name = 2;
  
  // display_name is the user's chosen display name
  string display_name = 3;
  
  // profile_image is the data associated with the user's profile image
  ProfileImage profile_image = 4;
}

// ProfileImage represents data associated with a user's profile image
message ProfileImage {

  // payload is a context based payload for the profile image data,
  // context is determined by the `source_type`
  string payload = 1;

  // source_type signals the image payload source
  SourceType source_type = 2;

  // image_type signals the image type and method of parsing the payload
  ImageType image_type =3;
  
  // SourceType are the predefined types of image source allowed
  enum SourceType {
    UNKNOWN_SOURCE_TYPE = 0;

    // RAW_PAYLOAD uses base64 encoded image data
    // `payload` must be set
    // `payload` is base64 encoded image data
    RAW_PAYLOAD = 1;

    // ENS_AVATAR uses the ENS record's resolver get-text-data.avatar data
    // The `payload` field will be ignored if ENS_AVATAR is selected
    // The application will read and parse the ENS avatar data as image payload data
    // The parent `ChatMessageIdentity` must have a valid `ens_name` set
    ENS_AVATAR = 2;
  }

  // ImageType is the type of profile image data
  enum ImageType {
    UNKNOWN_IMAGE_TYPE = 0;

    // RASTER_IMAGE_FILE is payload data that can be read as a raster image
    // examples: jpg, png, gif, webp file types
    RASTER_IMAGE_FILE = 1;

    // VECTOR_IMAGE_FILE is payload data that can be interpreted as a vector image
    // example: svg file type
    VECTOR_IMAGE_FILE = 2;

    // AVATAR is payload data that can be parsed as avatar compilation instructions
    AVATAR = 3;
  }

}
```

#### Implementation Recommendations

##### Identity Update

An `IdentityUpdate` is a concept representing the event of the application sending a `ChatMessageIdentity` in response to either:

- sending the user's first message in a chat topic
- sending a message to a chat topic after a change to the user identity
- sending a message to a chat topic after the expiry of the chat type `ChatMessageIdentity TTL` period

##### One-To-One Chat

This specification RECOMMENDS that the application only sends one `IdentityUpdate`, with no `ChatMessageIdentity TTL`.

##### Private Group Chat

This specification RECOMMENDS that the application only sends one `IdentityUpdate`, with not `ChatMessageIdentity TTL`, and once per new user joining the chat group.

##### Public Chat

 This specification RECOMMENDS that the application only sends one `IdentityUpdate`, with a `ChatMessageIdentity TTL` of 24 hours. 

##### Security

To preserve the privacy of the user, the `ProfileImage.payload` or ENS `get-text-data.avatar` field SHOULD NOT be or be parsed as a URL, an IPFS address or an IPNS address. Malicious actors could set their payload to an image URL and force all users that parse their `ProfileImage.payload` and log the IP address of users of selected topics.

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

### EmojiReaction

`EmojiReaction`s represents a user's "reaction" to a specific chat message. For more information about the concept of
emoji reactions see [Facebook Reactions](https://en.wikipedia.org/wiki/Facebook_like_button#Use_on_Facebook).

This specification RECOMMENDS that the UI/UX implementation of sending `EmojiReactions` requires only a single click
operation, as users have an expectation that emoji reactions are effortless and simple to perform.  

```protobuf
message EmojiReaction {
  // clock Lamport timestamp of the chat message
  uint64 clock = 1;

  // chat_id the ID of the chat the message belongs to, for query efficiency the chat_id is stored in the db even though the
  // target message also stores the chat_id
  string chat_id = 2;

  // message_id the ID of the target message that the user wishes to react to
  string message_id = 3;

  // message_type is (somewhat confusingly) the ID of the type of chat the message belongs to
  MessageType message_type = 4;

  // type the ID of the emoji the user wishes to react with
  Type type = 5;

  enum Type {
    UNKNOWN_EMOJI_REACTION_TYPE = 0;
    LOVE = 1;
    THUMBS_UP = 2;
    THUMBS_DOWN = 3;
    LAUGH = 4;
    SAD = 5;
    ANGRY = 6;
  }

 // whether this is a retraction of a previously sent emoji
  bool retracted = 6;
}
```

Clients MUST specify `clock`, `chat_id`, `message_id`, `type` and `message_type`.

This specification RECOMMENDS that the UI/UX implementation of retracting an `EmojiReaction`s requires only a single
click operation, as users have an expectation that emoji reaction removals are effortless and simple to perform.  

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

## Changelog

### Version 0.7

Released //TODO

- Added `ChatMessageIdentity` payload
- Marks `ChatMessage.ens_name` as DEPRECATED

### Version 0.5

Released [August 25, 2020](https://github.com/status-im/specs/commit/968fafff23cdfc67589b34dd64015de29aaf41f0)

- Added support for emoji reactions

### Version 0.4

Released [July 16, 2020](https://github.com/status-im/specs/commit/ad45cd5fed3c0f79dfa472253a404f670dd47396)

- Added support for images
- Added support for audio

### Version 0.3

Released [May 22, 2020](https://github.com/status-im/specs/commit/664dd1c9df6ad409e4c007fefc8c8945b8d324e8)

- Added language to include Waku in all relevant places

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
