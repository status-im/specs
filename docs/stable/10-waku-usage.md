---
permalink: /spec/10
parent: Stable specs
title: 10/WAKU-USAGE
---

# 10/WAKU-USAGE

> Version: 0.1
>
> Status: Stable
>
> Authors: Adam Babik <adam@status.im>, Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im>, Samuel Hawksby-Robinson <samuel@status.im> (alphabetical order)

- [Status Waku Usage Specification](#10waku-usage)
  - [Abstract](#abstract)
  - [Reason](#reason)
  - [Terminology](#terminology)
  - [Waku packets](#waku-packets)
  - [Waku node configuration](#waku-node-configuration)
  - [Status](#status)
  - [Rate limiting](#rate-limiting)
  - [Keys management](#keys-management)
    - [Contact code topic](#contact-code-topic)
    - [Partitioned topic](#partitioned-topic)
    - [Public chats](#public-chats)
    - [Group chat topic](#group-chat-topic)
    - [Negotiated topic](#negotiated-topic)
  - [Message encryption](#message-encryption)
  - [Message confirmations](#message-confirmations)
  - [Waku V1 extensions](#waku-v1-extensions)
    - [Request historic messages](#request-historic-messages)
      - [wakuext_requestMessages](#wakuext_requestmessages)
  - [Changelog](#changelog)
    - [Version 0.1](#version-01)

## Abstract

Status uses [Waku](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md) to provide
privacy-preserving routing and messaging on top of devP2P. Waku uses topics
to partition its messages, and these are leveraged for all chat capabilities. In
the case of public chats, the channel name maps directly to its Waku topic.
This allows anyone to listen on a single channel.

Additionally, since anyone can receive Waku envelopes, it relies on the
ability to decrypt messages to decide who is the correct recipient. This property
is not relied upon, and another secure transport layer is implemented on top of Waku.

## Reason

Provide routing, metadata protection, topic-based multicasting and basic
encryption properties to support asynchronous chat.

## Terminology

* *Waku node*: an Ethereum node with Waku V1 enabled
* *Waku network*: a group of Waku nodes connected together through the internet connection and forming a graph
* *Message*: decrypted Waku message
* *Offline message*: an archived envelope
* *Envelope*: encrypted message with metadata like topic and Time-To-Live

## Waku packets

| Packet Name          | Code | References |
| -------------------- | ---: | --- |
| Status               |    0 | [Status](#status), [WAKU-1](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#status) |
| Messages             |    1 | [WAKU-1](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#messages) |
| Batch Ack            |   11 | Undocumented. Marked for Deprecation |
| Message Response     |   12 | [WAKU-1](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#message-confirmations-update) |
| Status Update        |   22 | [WAKU-1](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#status-update) |
| P2P Request Complete |  125 | [4/WAKU-MAILSERVER](https://specs.status.im/spec/4) |
| P2P Request          |  126 | [4/WAKU-MAILSERVER](https://specs.status.im/spec/4), [WAKU-1](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#p2p-request) |
| P2P Messages         |  127 | [4/WAKU-MAILSERVER](https://specs.status.im/spec/4), [WAKU-1](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#p2p-message) |

## Waku node configuration

A Waku node must be properly configured to receive messages from Status clients.

Waku's Proof Of Work algorithm is used to deter denial of service and various spam/flood attacks against the Waku network. The sender of a message must perform some work which in this case means processing time. Because Status' main client is a mobile client, this easily leads to battery draining and poor performance of the app itself. Hence, all clients MUST use the following Waku node settings:
* proof-of-work requirement not larger than `0.002` for payloads less than 50,000 bytes
* proof-of-work requirement not larger than `0.000002` for payloads greater than or equal to 50,000 bytes
* time-to-live not lower than `10` (in seconds)

## Status

Handshake is a RLP-encoded packet sent to a newly connected peer. It MUST start with a Status Code (`0x00`) and follow up with items:
```
[
  [ pow-requirement-key pow-requirement ]
  [ bloom-filter-key bloom-filter ]
  [ light-node-key light-node ]
  [ confirmations-enabled-key confirmations-enabled ]
  [ rate-limits-key rate-limits ]
  [ topic-interest-key topic-interest ]
]
```

| Option Name             | Key    | Type     | Description | References |
| ----------------------- | ------ | -------- | ----------- | --- |
| `pow-requirement`       | `0x00` | `uint64` | minimum PoW accepted by the peer | [WAKU-1#pow-requirement](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#pow-requirement-field) |
| `bloom-filter`          | `0x01` | `[]byte` | bloom filter of Waku topic accepted by the peer | [WAKU-1#bloom-filter](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#bloom-filter-field) |
| `light-node`            | `0x02` | `bool`   | when true, the peer won't forward envelopes through the Messages packet. | `TODO` |
| `confirmations-enabled` | `0x03` | `bool`   | when true, the peer will send message confirmations | `TODO` |
| `rate-limits`           | `0x04` |          | See [Rate limiting](#rate-limiting) | [WAKU-1#rate-limits](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#rate-limits-field) |
| `topic-interest`        | `0x05` | `[10000][4]byte` | Topic interest is used to share a node's interest in envelopes with specific topics. It does this in a more bandwidth considerate way, at the expense of some metadata protection. Peers MUST only send envelopes with specified topics. | [WAKU-1#topic-interest](https://github.com/vacp2p/specs/blob/master/specs/waku/waku-1.md#topic-interest-field), [the theoretical scaling model](https://github.com/vacp2p/research/tree/dcc71f4779be832d3b5ece9c4e11f1f7ec24aac2/whisper_scalability) |

<!-- TODO Add `light-node` and `confirmations-enabled` links when https://github.com/vacp2p/specs/pull/128 is merged -->

## Rate limiting

In order to provide an optional very basic Denial-of-Service attack protection, each node SHOULD define its own rate limits. The rate limits SHOULD be applied on IPs, peer IDs, and envelope topics.

Each node MAY decide to whitelist, i.e. do not rate limit, selected IPs or peer IDs.

If a peer exceeds node's rate limits, the connection between them MAY be dropped.

Each node SHOULD broadcast its rate limits to its peers using `rate limits` in `status-options` via packet code `0x00` or `0x22`. The rate limits is RLP-encoded information:

```
[ IP limits, PeerID limits, Topic limits ]
```

`IP limits`: 4-byte wide unsigned integer
`PeerID limits`: 4-byte wide unsigned integer
`Topic limits`: 4-byte wide unsigned integer

The rate limits MAY also be sent as an optional parameter in the handshake.

Each node SHOULD respect rate limits advertised by its peers. The number of packets SHOULD be throttled in order not to exceed peer's rate limits. If the limit gets exceeded, the connection MAY be dropped by the peer.

## Keys management

The protocol requires a key (symmetric or asymmetric) for the following actions:
* signing & verifying messages (asymmetric key)
* encrypting & decrypting messages (asymmetric or symmetric key).

As asymmetric keys and symmetric keys are required to process incoming messages,
they must be available all the time and are stored in memory.

Keys management for PFS is described in [5/SECURE-TRANSPORT](https://specs.status.im/spec/5).

The Status protocols uses a few particular Waku topics to achieve its goals.

### Contact code topic

Contact code topic is used to facilitate the discovery of X3DH bundles so that the first message can be PFS-encrypted.

Each user publishes periodically to this topic. If user A wants to contact user B, she SHOULD look for their bundle on this contact code topic.

Contact code topic MUST be created following the algorithm below:
```golang
contactCode := "0x" + hexEncode(activePublicKey) + "-contact-code"

var hash []byte = keccak256(name)
var topicLen int = 4

if len(hash) < topicLen {
    topicLen = len(hash)
}

var topic [4]byte
for i = 0; i < topicLen; i++ {
    topic[i] = hash[i]
}
```

### Partitioned topic

Waku is broadcast-based protocol. In theory, everyone could communicate using a single topic but that would be extremely inefficient. Opposite would be using a unique topic for each conversation, however, this brings privacy concerns because it would be much easier to detect whether and when two parties have an active conversation.

Partitioned topics are used to broadcast private messages efficiently. By selecting a number of topic, it is possible to balance efficiency and privacy.

Currently, the number of partitioned topics is set to `5000`. They MUST be generated following the algorithm below:
```golang
var partitionsNum *big.Int = big.NewInt(5000)
var partition *big.Int = big.NewInt(0).Mod(publicKey.X, partitionsNum)

partitionTopic := "contact-discovery-" + strconv.FormatInt(partition.Int64(), 10)

var hash []byte = keccak256(partitionTopic)
var topicLen int = 4

if len(hash) < topicLen {
    topicLen = len(hash)
}

var topic [4]byte
for i = 0; i < topicLen; i++ {
    topic[i] = hash[i]
}
```

### Public chats

A public chat MUST use a topic derived from a public chat name following the algorithm below:
```golang
var hash []byte
hash = keccak256(name)

topicLen = 4
if len(hash) < topicLen {
    topicLen = len(hash)
}

var topic [4]byte
for i = 0; i < topicLen; i++ {
    topic[i] = hash[i]
}
```

<!-- NOTE: commented out as it is currently not used.  In code for potential future use. - C.P. Oct 8, 2019
### Personal discovery topic

 Personal discovery topic is used to ???

A client MUST implement it following the algorithm below:
```golang
personalDiscoveryTopic := "contact-discovery-" + hexEncode(publicKey)

var hash []byte = keccak256(personalDiscoveryTopic)
var topicLen int = 4

if len(hash) < topicLen {
    topicLen = len(hash)
}

var topic [4]byte
for i = 0; i < topicLen; i++ {
    topic[i] = hash[i]
}
```

Each Status Client SHOULD listen to this topic in order to receive ??? -->

### Group chat topic

Group chats does not have a dedicated topic. All group chat messages (including membership updates) are sent as one-to-one messages to multiple recipients.

### Negotiated topic

When a client sends a one to one message to another client, it MUST listen to their negotiated topic. This is computed by generating
a diffie-hellman key exchange between two members and taking the first four bytes of the `SHA3-256` of the key generated.

```golang

sharedKey, err := ecies.ImportECDSA(myPrivateKey).GenerateShared(
      ecies.ImportECDSAPublic(theirPublicKey),
      16,
      16,
)


hexEncodedKey := hex.EncodeToString(sharedKey)

var hash []byte = keccak256(hexEncodedKey)
var topicLen int = 4

if len(hash) < topicLen {
    topicLen = len(hash)
}

var topic [4]byte
for i = 0; i < topicLen; i++ {
    topic[i] = hash[i]
}
```

A client SHOULD send to the negotiated topic only if it has received a message from all the devices included in the conversation.

### Flow

To exchange messages with client B, a client A SHOULD:

- Listen to client's B Contact Code Topic to retrieve their bundle information, including a list of active devices
- Send a message on client's B partitioned topic
- Listen to the Negotiated Topic between A & B
- Once a message is received from B, the Negotiated Topic SHOULD be used

## Message encryption

Even though, the protocol specifies an encryption layer that encrypts messages before passing them to the transport layer, Waku protocol requires each Waku message to be encrypted anyway.

Public and group messages are encrypted using symmetric encryption and the key is created from a channel name string. The implementation is available in [`waku_generateSymKeyFromPassword`](https://github.com/status-im/status-go/tree/develop/_examples) JSON-RPC method of go-ethereum Whisper implementation.

One-to-one messages are encrypted using asymmetric encryption.

## Message confirmations

Sending a message is a complex process where many things can go wrong. Message confirmations tell a node that a message originating from it has been seen by its direct peers.

A node MAY send a message confirmation for any batch of messages received in a packet Messages Code (`0x01`).

A message confirmation is sent using Batch Acknowledge packet (`0x0b`) or Message Response packet (`0x0c`).

The Batch Acknowledge packet is followed by a keccak256 hash of the envelopes batch data (raw bytes).

The Message Response packet is more complex and is followed by a Versioned Message Response:
```
[ Version, Response]
```

`Version`: a version of the Message Response, equal to `1`,
`Response`: `[ Hash, Errors ]` where `Hash` is a keccak256 hash of the envelopes batch data (raw bytes) for which the confirmation is sent and `Errors` is a list of envelope errors when processing the batch. A single error contains `[ Hash, Code, Description ]` where `Hash` is a hash of the processed envelope, `Code` is an error code and `Description` is a descriptive error message.

The supported codes:
`1`: means time sync error which happens when an envelope is too old or created in the future (the root cause is no time sync between nodes).

The drawback of sending message confirmations is that it increases the noise in the network because for each sent message, a corresponding confirmation is broadcast by one or more peers. To limit that, both Batch Acknowledge packet (`0x0b`) and Message Response packet (`0x0c`) are not broadcast to peers of the peers, i.e. they do not follow epidemic spread.

In the current Status network setup, only `Mailservers` support message confirmations. A client posting a message to the network and after receiving a confirmation can be sure that the message got processed by the `Mailserver`. If additionally, sending a message is limited to non-`Mailserver` peers, it also guarantees that the message got broadcast through the network and it reached the selected `Mailserver`.

## Waku V1 extensions

### Request historic messages

Sends a request for historic messages to a `Mailserver`. The `Mailserver` node MUST be a direct peer and MUST be marked as trusted (using `waku_markTrustedPeer`).

The request does not wait for the response. It merely sends a peer-to-peer message to the `Mailserver` and it's up to `Mailserver` to process it and start sending historic messages.

The drawback of this approach is that it is impossible to tell which historic messages are the result of which request.

It's recommended to return messages from newest to oldest. To move further back in time, use `cursor` and `limit`.

#### wakuext_requestMessages

**Parameters**:
1. Object - The message request object:
   * `mailServerPeer` - `String`: `Mailserver`'s enode address.
   * `from` - `Number` (optional): Lower bound of time range as unix timestamp, default is 24 hours back from now.
   * `to` - `Number` (optional): Upper bound of time range as unix timestamp, default is now.
   * `limit` - `Number` (optional): Limit the number of messages sent back, default is no limit.
   * `cursor` - `String` (optional): Used for paginated requests.
   * `topics` - `Array`: hex-encoded message topics.
   * `symKeyID` - `String`: an ID of a symmetric key used to authenticate with the `Mailserver`, derived from the `Mailserver` password.

**Returns**:
`Boolean` - returns `true` if the request was sent.

The above `topics` is then converted into a bloom filter and then and sent to the `Mailserver`.

<!-- TODO: Clarify actual request with bloom filter to mailserver -->

## Changelog

### Version 0.1

Released [May 22, 2020](https://github.com/status-im/specs/commit/664dd1c9df6ad409e4c007fefc8c8945b8d324e8)

- Created document
- Forked from [3-whisper-usage](3-whisper-usage.md)
- Change to keep `Mailserver` term consistent
- Replaced Whisper references with Waku
- Added [Status options](#status) section
- Updated [Waku packets](#waku-packets) section to match Waku
- Added that `Batch Ack` is marked for deprecation 
- Changed `shh_generateSymKeyFromPassword` to `waku_generateSymKeyFromPassword`
  - [Exists here](https://github.com/status-im/status-go/blob/2d13ccf5ec3db7e48d7a96a7954be57edb96f12f/waku/api.go#L172-L175)
  - [Exists here](https://github.com/status-im/status-go/blob/2d13ccf5ec3db7e48d7a96a7954be57edb96f12f/eth-node/bridge/geth/public_waku_api.go#L33-L36)
- Changed `shh_markTrustedPeer` to `waku_markTrustedPeer`
  - [Exists here](https://github.com/status-im/status-go/blob/2d13ccf5ec3db7e48d7a96a7954be57edb96f12f/waku/api.go#L100-L108)
- Changed `shhext_requestMessages` to `wakuext_requestMessages`
  - [Exists here](https://github.com/status-im/status-go/blob/2d13ccf5ec3db7e48d7a96a7954be57edb96f12f/services/wakuext/api.go#L76-L139)