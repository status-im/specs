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

Messages are stored permanently by store nodes ([11/WAKU-MAILSERVER](/spec/11), or [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/)) for up to a certain configurable period of time, limited by the overall storage provided by a store node. Messages older than that period are no longer provided by store nodes, making it impossible for other nodes to request historical messages that go beyond that time range. This is especially problematic in the case of Status communities, where recently joined members of a community aren't able to request complete message histories of the community channels.

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
- [Message history archive index](#message-history-archive-index)
  - [WakuMessaegArchiveIndex](#wakumessagearchiveindex)
- [Creating message archive torrents](#creating-message-archive-torrents)
  - [Ensuring reproducible data pieces](#ensuring-reproducible-data-pieces)
    - [Example: without padding](#example-without-padding)
    - [Example: with padding](#example-with-padding)
- [Seeding message history archives](#seeding-message-history-archives)
- [Creating magnet links](#creating-magnet-links)
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
| Waku node            | An Waku node that implements [6/WAKU1](https://rfc.vac.dev/spec/6/), or a [10/WAKU2](https://rfc.vac.dev/spec/10/) node that implements [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/)|
| Store node           | A Waku node that implements [11/WAKU-MAILSERVER](/spec/11) or [13/WAKU2-STORE](https://rfc.vac.dev/spec/13/) respectively |
| Waku network         | A group of Waku nodes forming a graph, connected via [RLPx transport protocol](https://rfc.vac.dev/spec/6/#wire-specification) or [11/WAKU2-RELAY](https://rfc.vac.dev/spec/11/) respectively |
| Status user          | An Ethereum account that can be used in a Status consumer product, such as Status Mobile or Status Desktop |
| Status node          | A Status client run by a Status application |
| Community owner      | A Status user that owns the private key for a Status community |
| Community member     | A Status user that is part of a Status community, not owning the private key of the community |
| Community owner node | A Status node with message archive capabilities enabled, run by a community owner |
| Community member node| A Status node with message archive capabilities enabled, run by a community member |
| Live messages        | Waku messages received through the Waku network |
| BitTorrent client    | A program implementing the [BitTorrent](https://bittorrent.org) protocol |
| Torrent/Torrent file | A file containing metadata about data to be downloaded by BitTorrent clients |
| Magnet link          | A link encoding the metadata provided by a torrent file ([Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme)) |


## Supported platforms

Creating, distributing, downloading and unpacking community history archives SHOULD only be supported in Status desktop clients. Status mobile are out of scope for the MVP of this service. However we may consider bringing this service to Status Mobile in the future.


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

These assumptions are less than ideal and will be enhanced in future work. This [discussion](https://forum.vac.dev/t/status-communities-protocol-and-product-point-of-view/114) provides more information on that.

## Overview

The following is a high-level overview of the user flow and features this specification describes. For more detailed descriptions, read the dedicated sections in this specification.

### Serving community history archives

Community owner nodes go through the following (high level) process to provide community members with message histories (assumes community owner node is available 24/7):

1. Community owner creates a Status community (previously known as [org channels](https://github.com/status-im/specs/pull/151))
2. Community owner doesn't disable community history archive support (on by default but can be turned off as well - see [UI feature spec](https://github.com/status-im/feature-specs/pull/36))
3. A special type of channel to exchange metadata about the archival data is created, this channel should not be visible in the user interface
4. Community owner invites members and creates additional channels
5. Community owner node receives messages and stores them into local database
6. After 7 days, the community owner node exports and compresses last 7 days worth of messages from database and bundles it together with a [message archive index](#waku-message-archive-index) into a torrent, from which it then creates a magnet link ([Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme), [Extensions for Peers to Send Metadata Files](https://www.bittorrent.org/beps/bep_0009.html)) 
7. Community owner sends the magnet link created in step 6) to community members via special channel created in step 3) through the Waku network
8. Every subsequent 7 days, the steps 6) and 7) are repeated and the new message archive data is appended to the previously created message archive data 

### Serving archives for missed messages

If the community owner node goes offline (where "offline" means, the owner node's main process is no longer running), it MUST go through the following process:

1. Community owner node restarts
2. Community owner node requests messages from store nodes for the missed time range for all channels in their community
3. All missed messages are stored into community owner node's local message database
4. If 7 or more days have elapsed since the last message history torrent was created, the community owner node will perform step 6) and 7) of [Serving community history archives](#serving-community-history-archives) for every 7 days worth of messages in the missed time range (e.g. if the node was offline for 30 days, it will create 4 message history archives)

### Receiving community history archives

Community member nodes go through the following (high level) process to fetch and restore community message histories:

1. User joins community and becomes community member (see [org channels spec](https://github.com/status-im/specs/pull/151))
2. By joining a community, member nodes automatically subscribe to special channel for message archive metadata exchange provided by the community
3. Member node requests live message history (last 30 days) of special community channel from store nodes (in addition to last 7 days of message for all other community channels)
4. Member node receives waku message ([14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/)) that contains the metadata magnet link from the special channel
5. Member node extracts the magnet link from the waku message and passes it to torrent client
6. Member node downloads [message archive index](#message-history-archive-index) file and determines which message archives are not downloaded yet (all or some)
7. Member node fetches missing message archive data via torrent
8. Member node unpacks and decompresses message archive data to then hydrate its local database, deleting any messages for that community that the database previously stored in the same time range as covered by the message history archive

## Storing live messages

Community owner nodes MUST store live messages as [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/). This is required to provide confidentiality, authenticity, and integrity of message data distributed via the BitTorrent layer, and later validated by Status nodes when they unpack message history archives.

Community owner nodes SHOULD remove those messages from their local databases once they are older than 30 days and after they have been turned into message archives and distributed to the BitTorrent network.

## Exporting messages for bundling

Community owner nodes export messages from their local database for creating and bundling history archives using the following criteria:

- Messages to be exported MUST have a `contentTopic` that match any of the topics of the community channels
- Messages to be exported MUST have a `timestamp` that lies within a time range of 7 days

The `timestamp` is determined by the context in which the community owner node attempts to create a message history archives as described below:

1. The community owner node attempts to create an archive periodically for the past seven days (including the current day). In this case, the `timestamp` has to lie within those 7 days.
2. The community owner node has been offline (owner node's main process has stopped and needs restart) and attempts to create archives for all the live messages it has missed since it went offline. In this case, the `timestamp` has to lie within the day the latest message was received and the current day.

Exported messages MUST be restored as [14/WAKU2-MESSAGE](https://rfc.vac.dev/spec/14/) for bundling. Waku messages that are older than 30 days and have been exported for bundling can be removed from the community owner node's database (community owner nodes still maintain a database of application messages).

## Message history archives

Message history archives are represented as `WakuMessageArchive` and created from waku messages exported from the local database. Message history archives are implemented using the following protocol buffer.

### WakuMessageHistoryArchive

The `from` field SHOULD contain a timestamp of the time range's lower bound.

The `to` field SHOULD contain a timestamp of the time range's the higher bound.

The `contentTopic` field MUST contain a list of all communiity channel topics.

The `messages` field MUST contain all messages that belong into the archive given its `from`, `to` and `contentTopic` fields.

The `padding` field MUST contain the amount of zero bytes needed so that the overall byte size of the protobuf encoded `WakuMessageArchive` is a multiple of the `pieceLength` used to divide the message archive data into pieces, as explained in [creating message archive torrents](#creating-message-archive-torrents).

```
syntax = "proto3"

message WakuMessageArchiveMetadata {
  uint8 version = 1
  uint64 from = 2
  uint64 to = 3
  repeated string contentTopic = 4
}

message WakuMessageArchive {
  uint8 version = 1
  WakuMessageArchiveMetadata metadata = 2
  repeated WakuMessage messages = 3 // `WakuMessage` is provided by 14/WAKU2-MESSAGE
  bytes padding = 4
}
```

## Message history archive index

Community owner nodes MUST provide message archives for the entire community history. Each individual archive only contains a subset of the complete history, that is, data for a time range of seven days, and all message history archives are concatenated into a single file as byte string (see [Ensuring reproducible data pieces](#ensuring-reproducible-data-pieces)).

Community owner nodes MUST create a message history archive index (`WakuMessageArchiveIndex`) with metadata that allows receiving nodes to only fetch the message history archives they are interested in.

### WakuMessageArchiveIndex

A `WakuMessageArchiveIndex` is a map where the key is the KECCAK-256 hash of the `WakuMessageArchiveIndexMetadata` derived from a 7-day archive and the value is an instance of that `WakuMessageArchiveIndexMetadata` corresponding to that archive.

The `offset` field MUST contain the position at which the message history archive starts in the byte string of the total message archive data. This MUST be the sum of the length of all previously created message archives in bytes (see [Creating message archive torrents](#creating-message-archive-torrents)).

```
syntax = "proto3"

message WakuMessageArchiveIndexMetadata {
  uint8 version = 1
  WakuMessageArchiveMetadata metadata = 2
  uint64 offset = 3
  uint64 num_pieces = 4
}

message WakuMessageArchiveIndex {
  map<string, WakuMessageArchiveIndexMetadata> archives = 1
}
```

The community owner node MUST update the `WakuMessageArchiveIndex` every time it creates one or more `WakuMessageArchive`s and bundle it into a new torrent (**TODO: see section**).
For every created `WakuMessageArchive`, there MUST be a `WakuMessageArchiveIndexMetadata` entry in the `archives` field `WakuMessageArchiveIndex`.

## Creating message archive torrents

Community owner nodes MUST create a torrent file ("torrent") containing metadata to all message history archives. To create a torrent file, and later serve the message archive data in the BitTorrent network, community owner nodes MUST store the necessary data in dedicated files on the file system.

A torrent's source folder MUST contain the following two files:

- `data` - Contains all protobuf encoded message history archives concatenated in ascending order
- `index` - Contains the protobuf encoded message history archive index

Community owner nodes SHOULD store these files in a dedicated folder that is identifiable via the community id.

### Ensuring reproducible data pieces

The community owner node MUST ensure that the byte string resulting from the protobuf encoded `data` is equal to the byte string `data` from the previously generated message archive torrent, plus the data of the latest 7 days worth of messages encoded as `WakuMessageArchive`. Therefore, the size of `data` grows every seven days as it's append only.

The community owner nodes also MUST ensure that the byte size of every individual `WakuMessageArchive` encoded protobuf is a multiple of `pieceLength: ???` (**TODO**) using the `padding` field. If the last piece of a message archive has fewer bytes than `pieceLength`, it MUST be filled with zero bytes until it has the size `pieceLength`.

This is necessary because message history archive data will be split into pieces of `pieceLength` when the torrent file is created, and the SHA1 hash of every piece is then stored in the torrent file and later used by other nodes to request the data for each individual data piece.

By fitting message archives into a multiple of `pieceLength` and ensuring they fill possible remainding space with zero bytes, community owner nodes prevent the **next** message archive to occupy that remainding space of the last piece, which will result in a different SHA1 hash for that piece.

#### **Example: Without padding**

Let `WakuMessageArchive` "A1" be of size 20 bytes:

```
 0 11 22 33 44 55 66 77 88 99
10 11 12 13 14 15 16 17 18 19 
```

With a `pieceLength` of 10 bytes, A1 will fit into `20 / 10 = 2` pieces:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
```

#### **Example: With padding**

Let `WakuMessageArchive` "A2" be of size 21 bytes:

```
 0 11 22 33 44 55 66 77 88 99
10 11 12 13 14 15 16 17 18 19
20
```

With a `pieceLength` of 10 bytes, A2 will fit into `21 / 10 = 2` pieces. The remainder will introduce a third piece:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
20                            // piece[2] SHA1: 0x789
```

The next `WakuMessageArchive` "A3" will be appended ("#3") to the existing data and occupy the remainding space of the third data piece. The piece at index 2 will now produce a different SHA1 hash:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
20 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[2] SHA1: 0xeef
#3 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[3]
```

By filling up the remainding space of the third piece with A2 using its `padding` field, it is guaranteed that its SHA1 will stay the same:

```
 0 11 22 33 44 55 66 77 88 99 // piece[0] SHA1: 0x123
10 11 12 13 14 15 16 17 18 19 // piece[1] SHA1: 0x456
20  0  0  0  0  0  0  0  0  0 // piece[2] SHA1: 0x999
#3 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[3]
#3 #3 #3 #3 #3 #3 #3 #3 #3 #3 // piece[4]
```

## Seeding message history archives

The community owner node MUST seed the [generated torrent](#creating-message-archive-torrents) until a new message history archive is created.

The community owner node SHOULD NOT seed torrents for older message history archives. Only one torrent at a time should be seeded.

## Creating magnet links

Once a torrent file for a message archive is created, the community owner node MUST derive a magnet link following the [Magnet URI scheme](https://en.wikipedia.org/wiki/Magnet_URI_scheme) using the underlying BitTorrent protocol client.

## Message archive distribution

Message archives are available via the BitTorrent network as they are being [seeded by the community owner node](#seeding-message-history-archives).
Other community member nodes will download the message archives from the BitTorrent network once they receive a magnet link that contains a message archive index.

The community owner node MUST send magnet links containing message archives and the message archive index to a special community channel. The topic of that special channel follows the following format:

```
/{application-name}/{version-of-the-application}/{content-topic-name}/{encoding}
```

All messages sent with this topic MUST be instances of `ApplicationMetadataMessage` ([6/PAYLOADS](/specs/6-payloads)) with a `payload` of `CommunityMessageArchiveIndex`.

Only the community owner MAY post to the special channel. Other messages on this specified channel MUST be ignored by clients.
Community members MUST NOT have permission to send messages to the special channel.
However, community member nodes MUST subscribe to special channel to receive waku messages containing magnet links for message archives.

## Canonical message histories

Only community owners are allowed to distribute messages with magnet links via the special channel for magnet link exchange. Community members MUST NOT be allowed to post any messages to the special channel.

Status nodes MUST ensure that any message that isn't signed by the community owner in the special channel is ignored.

Since the magnet links are created from the community owner node's database (and previously distributed archives), the message history provided by the community owner becomes the canonical message history and single source of truth for the community.

Community member nodes MUST replace messages in their local databases with the messages extracted from archives within the same time range. Messages that didn't receive the community owner node MUST be removed and are no longer part of the message history of interest, even if it already existed in a community member node's database.


## Fetching message history archives

Generally, fetching message history archives is a tree step process:

1. Receive message archive index magnet link as described in [Message archive distribution], download `index` file from torrent, then determine which message archives to download
3. Download individual archives

Community member nodes subscribe to the special channel that community owner nodes publish magnet links for message history archives to. There are two scenarios in which member nodes can receive such a magnet link message from the special channel:

1. The member node receives it via live messages, that is, messages that are relayed by store nodes 
2. The member node requests messages for a time range of up to 30 days from store nodes (this is the case when a new community member joins a community)

### Downloading message archives
When member nodes receive a message with a `CommunityMessageHistoryArchive` ([6/PAYLOADS](/spec/6#communitymessagearchive)) from the aforementioned channnel, they MUST extract the `magnet_uri` and pass it to their underlying BitTorrent client so they can fetch the latest message history archive index, which is the `index` file of the torrent (see [Creating message archive torrents](#creating-message-archive-torrents)).

Due to the nature of distributed systems, there's no guarantee that a received message is the "last" message. This is especially true when member nodes request historical messages from store nodes. 

Therefore, member nodes MUST wait for 20 seconds after receiving the last `CommunityMessageArchive` before they start extracting the magnet link to fetch the latest archive index.

Once a message history archive index is downloaded and parsed back into `WakuMessageArchiveIndex`, community member nodes use a local lookup table to determine which of the listed archives are missing using the KECCAK-256 hashes stored in the index.

For this lookup to work, member nodes MUST store the KECCAK-256 hashes of the `WakuMessageArchiveIndexMetadata` provided by the `index` file for all of the message history archives that have been downlaoded in their local database.

Given a `WakuMessageArchiveIndex`, member nodes can access individual `WakuMessageArchiveIndexMetadata` to download individual archives.

Community member nodes MUST choose one of the following options:

1. **Download all archives** - Request and download all data pieces for `data` provided by the torrent (this is the case for new community member nodes that haven't downloaded any archives yet)
2. **Download only the latest archive** - Request and download all pieces starting at the `offset` of the latest `WakuMessageArchiveIndexMetadata` (this the case for any member node that already has downloaded all previous history and is now interested in only the latst archive)
3. **Download specific archives** - Look into `from` and `to` fields of every `WakuMessageArchiveIndexMetadata` and determine the pieces for archives of a specific time range (can be the case for member nodes that have recently joined the network and are only interested in a subset of the complete history)

## Storing historical messages

When message archives are fetched, community member nodes MUST unwrap the resulting `WakuMessage` instances into `ApplicationMetadataMessage` instances and store them in their local database.
Community member nodes SHOULD NOT store the wrapped `WakuMessage` messages.

All message within the same time range MUST be replaced with the messages provided by the message history archive.

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
