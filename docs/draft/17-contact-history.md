---
permalink: /draft/17
title: 7/CONTACT-HISTORY
parent: Draft specs
layout: default
---

# 7/CONTACT-HISTORY

> Version: 0.1
>
> Status: Draft
>
> Authors: Iuri Matias <iuri@status.im>, Jonathan Rainville <jonathanrainville@status.im>, Richard Ramos <richard@status.im>
>

## Table of Contents

- [Abstract](#abstract)
- [Modifications to Message Format](#Modifications-to-Message-Format)
- [Advertising Capabilities](#Advertising-Capabilities)
- [Requesting Chat history](#Requesting-Chat-history)
- [Requesting 1 on 1 chat history](#Requesting-1-on-1-chat-history)
- [Requesting a group chat history](#Requesting-a-group-chat-history)
- [Requesting a public chat history](#Requesting-a-public-chat-history)
- [Requesting a community chat history](#Requesting-a-community-chat-history)

## Abstract

When Store Nodes (aka Mailservers) are unavailable or faulty, short term history is lost. This spec proposes a solution for which short term history can be recovered through trusted contacts without relying on Store Nodes. It is not meant to replace Store Nodes or other solutions, but instead to complement them.

### Modifications to Message Format

The `ApplicationMetadataMessage` is modified to include 3 new Types:

```
message ApplicationMetadataMessage {
  // Signature of the payload field
  bytes signature = 1;
  // This is the encoded protobuf of the application level message, i.e ChatMessage
  bytes payload = 2;

  // The type of protobuf message sent
  Type type = 3;

  enum Type {
    // ...
    HISTORY_CAPABILITY = 42;
    HISTORY_REQUEST = 43;
    HISTORY_REPLY = 44;
  }
}
```

## Advertising Capabilities

- When a contact request is sent the sender MAY advertise its capabilities
- When a contact request is accepted the receiver MAY advertises its own capabilities

```
HistoryCapability {
  uint64 clock = 1;
  string contactChatId = 2
  map<string, int64> capability = 3;
}
```

`HistoryCapability.capability` has the mapping between chatIds and time in seconds. This indicates the contact is willing to send the chat history for last X seconds for that particular chatId.

```
{
  "0x123": -1, // all available chat history
  "0x234": 3600, // last 1h
  "0x345": 300, // last 5 minutes
}
```

If no capabilities are advertised, it MUST be assumed that contact will not relay contact history.

When a user changes its capabilities, a change signal MUST be sent to contacts.

## Requesting Chat history

The client chooses one of his online contacts at random that indicated capability to send the history for the target chat or community.

**Request**

A `ChatMessage` with the ContentType `HISTORY` and the timestamp of the last known message in the 1 on 1 chat.

```
HistoryRequest {
  uint64 clock = 1;
  string contactChatId = 2
  uint64 requestedChatId = 3;
  uint64 requestedLastTimestamp = 4;
}
```

**Response**

```
HistoryResponse {
  uint64 clock = 1;
  string contactChatId = 2
  uint64 requestedChatId = 3;
  string message = 4;
}
```

`HistoryResponse.message` contains the encoded original message.
A `HistoryResponse` is sent for each message in the requested time range.

## Requesting 1 on 1 chat history

In this case, only 1 contact will be a match. If the contact has not advertised intention to send the chat history, no request should be sent.

## Requesting a group chat history

The client chooses one of the online contacts that are members in the group chat at random to request the group history and that has advertised intention to send the chat history of this group chat.
If no history is received during time X, then the client can choose another contact to request the history

## Requesting a public chat history

The client chooses one of the online contacts that displayed intentions to send chat history for the public chat in question.


## Requesting a community chat history

The client chooses one of the online contacts that displayed intentions to send chat history for the community in question.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
