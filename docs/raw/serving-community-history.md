---
permalink: /raw/17
title: 17/SERVING-COMMUNITY-HISTORY-MVP
parent: Raw specs
layout: default
---

# 17/SERVING-COMMUNITY-HISTORY-MVP

> Version: 0.1
>
> Status: Raw
>
> Authors: Pascal Precht <pascal@status.im>

## Abstract

Messages are stored permanently by store nodes ([11/WAKU-MAILSERVER](/spec/11), or [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/)) for up to 30 days. Messages older than that are no longer provided by store nodes, making it impossible for other nodes to request historical messages older than that. This is especially problematic in the case of Status communities, where recently joined members of a community aren't able to request complete message histories of the community channels.

This specification describes how community owners archive historical message data of their communities beyond the time range limit provided by store nodes using the [BitTorrent](https://bittorrent.org) protocol. It also describes how the archives are distributed to community members via the Status network, so they can fetch them and get access to a complete message history.

## Table of Contents

- [Abstract](#abstract)
- [Table of Contents](#table-of-contents)
- [Terminology](#terminology)
- [Supported platforms](#supported-platforms)
- [Requirements / Assumptions](#requirements--assumptions)
- [Overview](#overview)
  - [Serving community history archives](#serving-community-history-archives)
  - [Serving archives for missed messages](#serving-archives-for-missed-messages)
  - [Receiving community history archives](#receiving-community-history-archives)
- [Storing live messages](#storing-live-messages)
- [Exporting messages for bundling](#exporting-messages-for-bundling)
- [Message history archives](#message-history-archives)
  - [WakuMessageHistoryArchive](#wakumessagehistoryarchive)
- [Creating magnet links](#creating-magnet-links)
- [Bundling history archives into archive indices](#bundling-history-archives-into-archive-indices)
  - [WakuMessageArchiveMetadata](#wakumessagearchivemetadata)
- [Message archive distribution](#message-archive-distribution)
- [Canonical message histories](#canonical-message-histories)
- [Fetching messsage archives](#fetching-message-archives)
  - [Downloading message archive indices](#downloading-message-archive-indices)
  - [Downloading individual archives](#downloadinng-individual-archives)
- [Storing historical messages](#storing-historical-messages)
- [Considerations](#considerations)
  - [Bandwidth consumption](#bandwidth-consumption)
  - [Multiple community owners](#multiple-community-owners)

## Terminology

The following terminology is used throughout this specification:

| Name                 | References |
| -------------------- | --- |
| Waku node            | An Ethereum node with Waku V1 enabled, or a [10/WAKU2](https://rfc.vac.dev/spec/10/) node that implements [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/)|
| Store node           | A Waku node that implements [11/WAKU-MAILSERVER](/spec/11) or [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) respectively |
| Waku network         | A group of Waku nodes connected through the internet connection and forming a graph |
| Community owner      | A Status user that owns a Status community |
| Community member     | A Status user that is part of a Status community |
| Community owner node | A Status node with message archive capabilities enabled, run by a community owner |
| Community member node| A Status node with message archive capabilities enabled, run by a community member |
| Live messages        | Waku messages received through the Waku network |


## Supported platforms

Creating, distributing, downloading and unpacking community history archives SHOULD only be supported in desktop clients. Mobile clients SHOULD NOT implement this functionality due to bandwidth and storage limitations.

## Requirements / Assumptions

This specification has the following assumptions:

- Store nodes are available 24/7, ensuring constant live message availability
- Store nodes have enough storage to persist historical messages for up to 30 days
- No store nodes have storage to persist historical messages older than 30 days
- All nodes are honest
- The network is reliable

Furthermore, it assumes that:

- Community owner nodes have enough storage to persist historical messages older than 30 days
- Community owner nodes provide archives with historical messages **at least** every 30 days
- Community owner nodes receive all community messages
- Community owner nodes are honest

## Overview

The following is a high-level overview of the user flow and features this specification describes.

### Serving community history archives

Community owner nodes go through the following (high level) process to provide community members with message histories (assumes community owner node is available 24/7):

1. Community owner creates a Status community
2. Community owner enables community history archive support
3. A special type of channel for distributing magnet links ([Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme), [Extensions for Peers to Send Metadata Files](https://www.bittorrent.org/beps/bep_0009.html)) is created
4. Community owner invites members and creates additional channels
5. Community owner node receives messages and stores them into local database
6. After 7 days, the community owner node exports and compresses last 7 days worth of messages from database and creates a magnet link from that data via torrent client
7. Community owner node creates message archive index and bundles the previously generated magnet link into it
8. Community owner node creates magnet link from index and distributes it to community members via special channel created in step 2) through the Waku network
9. Every subsequent 7 days, the step 6) - 8) are repeated and bundled with the previously distributed archive

### Serving archives for missed messages

If the community owner node goes offline, it MUST go through the following process:

1. Community owner node restarts
2. Community owner node requests messages from store nodes for the missed time range
3. Missed messages are stored into local database
4. Community owner node creates message archive and magnet link for missed messages
5. Community oowner node creates new message archive index bundled with newly created magnet link
6. The new index is is distributed to community members via special channel through the Waku network

### Receiving community history archives

Community member nodes go through the following (high level) process to fetch and restore community message histories:

1. User joins community and becomes community member
2. By joining a community, member nodes automatically subscribe to special magnet link channel provided by the community
3. Member node requests message history (last 30 days) of community channels from store nodes
4. Member node receives magnet link message from store nodes
5. Member node extracts magnet link from message and passes it to torrent client
6. Torrent client downloads latest message archive index via magnet link
7. Member node fetches missing archives via torrent
8. Member node unpacks and decompresses message archive data to then hydrate its local database

## Storing live messages

Community owner nodes MUST store live messages as [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/). This is required to provide confidentiality, authenticity, and integrity of message data distributed via the BitTorrent layer, and later validated by Status nodes when they unpack message history archives.

Community owner nodes SHOULD remove those messages from their local databases after they have been turned into archives and distributed to the BitTorrent network.

## Exporting messages for bundling

Community owner nodes export messages from their local database for creating and bundling history archives using the following criteria:

- Messages to be exported MUST have a `contentTopic` that matches any of the topics of the community channels
- Messages to be exported MUST have a `timestamp` that lies within a given time range

The range for the `timestamp` depends on the context in which the community owner node attempts to create a history archive. This can be one of the following:

1. The community owner node attempts to create an archive periodically for the past seven days (including the current day). In this case, the `timestamp` has to lie within the day the last time an archive was created and the current day.
2. The community owner node has been offline and attempts to create an archive for all the live messages it has missed since it went offline. In this case, the `timestamp` has to lie within the day the latest message was received and the current day.

Exported messages MUST be restored as [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/) for bundling. Waku messages that have been exported for bundling can now be removed from the community owner node's database (community owner nodes still maintain a database of application messages).

## Message history archives

Message history archives are represented as `WakuMessageArchive` and created from messages exported from the local database. Message history archives are implemented using the following protocol buffer.

### WakuMessageHistoryArchive

The `from` field SHOULD contain a timestamp of the time range's lower bound.

The `to` field SHOULD contain a timestamp of the time range's the higher bound.

The `contentTopic` field MUST contain the same `contentTopic` that the archive's `messages` have.

The `messages` field MUST contain all messages that belong into the archive given its `from`, `to` and `contentTopic` fields.

```
syntax = "proto3"

message WakuMessageArchive {
  uint64 from = 1
  uint64 to = 2
  string contentTopic = 3
  repeated WakuMessage messages = 4 // `WakuMessage` is provided by 14/WAKU2-MESSAGE
}
```

## Creating magnet links

Once a message archive is created, the community owner node MUST derive a magnet link following the [Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme) using the underlying BitTorrent protocol client.

The `dn` parameter ("display name") in the resulting magnet link MAY be optional.

The resulting magnet link MUST be bundled into a `WakuMessageArchiveIndex`, which is then later distributed to other Status nodes.

## Bundling history archives into archive indices

Community owner nodes MUST provide message archives for the entire community history. However, each individual archive only contains a subset of the complete history, that is, either data for a time range of seven days, or, a time range in which the node was offline. Therefore, message history archives need to be bundled into a `WakuMessageArchiveIndex`, which later distributed via the Waku network and allows receiving nodes to fetch archives for individual time ranges.

A `WakuMessageArchiveIndex` is a map where the key is the KECCAK-256 hash of the magnet link derived from the previously created archive and the value an instance of `WakuMessageArchiveMetadata`, which is also derived from the previously created message archive.

### WakuMessageArchiveMetadata

The `from` field SHOULD contain a timestamp of the time range's lower bound.

The `to` field SHOULD contain a timestamp of the time range's the higher bound.

The `contentTopic` field MUST contain the same `contentTopic` that the archive's `messages` have.

The `magnet_uri_hash` field MUST contain the KECCAK-256 hash of the magnet link created from the `WakuMessageArchive`.

The `magnet_uri` field MUST contain the magnet link created from the `WakuMessageArchive`.

```
syntax = "proto3"

message WakuMessageArchiveMetadata {
  uint64 from = 1
  uint64 to = 2
  string contentTopic = 3
  string magnet_uri_hash = 4
  string magnet_uri = 5
}

message WakuMessageArchiveIndex {
  map<string, WakuMessageArchiveMetadata> archives = 1
}
```

The community owner node MUST create a `WakuMessageArchiveIndex` every time it creates a new `WakuMessageArchive`.

For every created `WakuMessageArchive`, there MUST be a `WakuMessageArchiveMetadata` entry in the index map.

The the community owner node MUST derive a magnet link from the newly created `WakuMessageArchiveIndex` so it can be distributed to community member nodes.

## Message archive distribution

Message archives are available via the BitTorrent network as soon as magnet links for them have been created.
Other community member nodes will download the message archives from the BitTorrent network once they receive a magnet link that contains a message archive index.

The community owner node MUST send magnet links containing message archive indices to a special community channel. The topic of that special channel consists of the community id followed by the string `-archives`:

```
{community_id}-archives
```

All messages sent with this topic MUST be instances of `ApplicationMetadataMessage` ([6/PAYLOADS](/specs/6-payloads)) with a `payload` of `CommunityMessageArchiveIndex`.

Only the community owner has permission to send messages with this topic.
Community members MUST NOT have permission to send messages with this topic.
However, community member nodes MUST subscribe to this topic to receive message archive indices.

## Canonical message histories

Only community owners are allowed to distribute messages with magnet links via the magnet link channel. Community members MUST NOT be allowed to distribute magnet links. Since the magnet links are created from the community owner node's database (and previously distributed archives), the message history provided by the community owner becomes the canonical message history and single source of truth for the community.

Community member nodes MUST replace messages in their local databases with the messages extracted from archives within the same time range. Messages that didn't receive the community owner node MUST be removed and are no longer part of the message history of interest, even if it already existed in a community member node's database.


## Fetching message archives

Generally, fetching message archives is a tree step process:

1. Receive message archive index signal, download index, then determine which message archives to download
3. Download individual archives

Community member nodes subscribe to the read-only channel that community owner nodes publish message archive indices to. There are two scenarios in which member nodes can receive message archive index messages:

1. The member node receives it via live messages, that is, messages that are relayed by store nodes 
2. The member node requests messages for a time range of up to 30 days from store nodes (this is the case when a new community member joins a community)

### Downloading message archive indices
When member nodes receive a message with a `CommunityMessageArchiveIndex` ([6/PAYLOADS](/specs/6-payloads)) from the aforementioned channnel, they MUST extract the `magnet_uri` and pass it to their underlying BitTorrent client so they can fetch the latest message archive index.

Due to the nature of distributed systems, there's no guarantee that a received message is the "last" message. This is especially true when member nodes request historical messages from store nodes. 

Therefore, member nodes MUST wait for 20 seconds after receiving the last `CommunityMessageArchiveIndex` before they start extracting the magnet link to fetch the latest archive index.

### Downloading individual archives
Once a message archive index is downloaded, community member nodes use a local lookup table to determine which of the listed archives are missing. For this lookup to work, member nodes MUST store the KECCAK-256 hashes of the magnet links for archives they've downloaded.

Given a list of magnet links provided by the index to download individual archives, member nodes can:

1. **Download all archives** - Extract each magnet link in the index and pass them to the underlying BitTorrent client (this is the case for new community member nodes that haven't downloaded any archives yet)
2. **Download only the latest archive** - Extract only the newest magnet link and pass it to the BitTorrent client (this the case for any member node that already has downloaded all previous history and is now interested in only the latst archive)
3. **Download specific archives** - Look into `from` and `to` fields of every `WakuMessageArchiveMetadata` and only extract magnet links for archives of a specific time range (can be the case for member nodes that have recently joined the network and are only interested in a subset of the complete history)


## Storing historical messages

When message archives are fetched, community member nodes MUST unwrap the resulting `WakuMessage` instances into `ApplicationMetadataMessage` instances and store them in their local database.
Community member nodes SHOULD NOT store the wrapped `WakuMessage` messages.

Already stored messages with the same `id` or `clock` value MUST be replaced with messages extracted from archives, if both of these values are equal.

Community members nodes MUST ignore the expiration state of each archive message.

## Considerations

The following are things to cosider when implementing this specification.

### Bandwidth consumption

Community member nodes will download the latest archive they've received from the archive index, which includes messages from the last seven days. Assuming that community members nodes were online for that time range, they have already downloaded that message data and will now download an archive that contains the same.

This means there's a possibility member nodes will download the same data at least twice.

### Multiple community owners

It is possible for community owners to export the private key of their owned community and pass it to other users so they become community owners as well. This means, it's possible for multiple owners to exist.

This might conflict with the assumption that the community owner node serves as a single source of thruth. Multiple owners can have different message histories.

Not only will multiple owners multiply the amount of archive index messages being distributed to the network, they might also contain different sets of magnet links and their corresponding hashes.

Even if just a single message is missing in one of the histories, the hashes presented in archive indices will look completely different, resulting in the community member node to download the corresponding archive (which might be identical to an archive that was already downloaded, except for that one message).


## Changelog

### Version 0.1

Released [](https://github.com/status-im/specs/commit/)

- Initial version

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
