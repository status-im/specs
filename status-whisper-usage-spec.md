# Status Whisper Usage Specification

> Version: 0.2
>
> Status: Stable
>
> Authors: Adam Babik <adam@status.im>, Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

- [Status Whisper Usage Specification](#status-whisper-usage-specification)
  - [Abstract](#abstract)
  - [Reason](#reason)
  - [Terminology](#terminology)
  - [Whisper packets](#whisper-packets)
  - [Whisper node configuration](#whisper-node-configuration)
  - [Handshake](#handshake)
  - [Rate limiting](#rate-limiting)
  - [Keys management](#keys-management)
    - [Contact code topic](#contact-code-topic)
    - [Partitioned topic](#partitioned-topic)
    - [Public chats](#public-chats)
    - [Generic discovery topic](#generic-discovery-topic)
    - [One-to-one topic](#one-to-one-topic)
    - [Group chat topic](#group-chat-topic)
  - [Message encryption](#message-encryption)
  - [Message confirmations](#message-confirmations)
  - [Whisper V6 extensions](#whisper-v6-extensions)
    - [Request historic messages](#request-historic-messages)
      - [shhext_requestMessages](#shhextrequestmessages)

## Abstract

Status uses [Whisper](https://eips.ethereum.org/EIPS/eip-627) to provide
privacy-preserving routing and messaging on top of devP2P. Whisper uses topics
to partition its messages, and these are leveraged for all chat capabilities. In
the case of public chats, the channel name maps directly to its Whisper topic.
This allows allows anyone to listen on a single channel.

Additionally, since anyone can receive Whisper envelopes, it relies on the
ability to decrypt messages to decide who is the correct recipient. We do
however not rely on this property, but instead implement another secure
transport layer on top of Whisper.

Finally, we use an extension of Whisper to provide the ability to do offline
messaging.

## Reason

Provide routing, metadata protection, topic-based multicasting and basic
encryption properties to support asynchronous chat.

## Terminology

* *Whisper node*: an Ethereum node with Whisper V6 enabled (in the case of geth, it's `--shh` option)
* *Whisper network*: a group of Whisper nodes connected together through the internet connection and forming a graph
* *Message*: decrypted Whisper message
* *Offline message*: an archived envelope
* *Envelope*: encrypted message with metadata like topic and Time-To-Live

## Whisper packets

| Packet Name | Code | EIP-627 | References |
| --- | --: | --- | --- |
| Status | 0 | ✔ | [Handshake](#handshake) |
| Messages | 1 | ✔ | [EIP-627](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-627.md) |
| PoW Requirement | 2 | ✔ | [EIP-627](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-627.md) |
| Bloom Filter | 3 | ✔ | [EIP-627](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-627.md) |
| Batch Ack | 11 | 𝘅 | Undocumented |
| Message Response | 12 | 𝘅 | Undocumented |
| P2P Sync Request | 123 | 𝘅 | Undocumented |
| P2P Sync Response | 124 | 𝘅 | Undocumented |
| P2P Request Complete | 125 | 𝘅 | [Status Whisper Mailserver Spec](status-whisper-mailserver-spec.md) |
| P2P Request | 126 | ✔ | [Status Whisper Mailserver Spec](status-whisper-mailserver-spec.md) |
| P2P Messages | 127 | ✔/𝘅 (EIP-627 supports only single envelope in a packet) | [Status Whisper Mailserver Spec](status-whisper-mailserver-spec.md) |


## Whisper node configuration

If you want to run a Whisper node and receive messages from Status clients, it must be properly configured.

Whisper's Proof Of Work algorithm is used to deter denial of service and various spam/flood attacks against the Whisper network. The sender of a message must perform some work which in this case means processing time. Because Status' main client is a mobile client, this easily leads to battery draining and poor performance of the app itself. Hence, all clients MUST use the following Whisper node settings:
* proof-of-work requirement not larger than `0.002`
* time-to-live not lower than `10` (in seconds)

## Handshake

Handshake is a RLP-encoded packet sent to a newly connected peer. It MUST start with a Status Code (`0x00`) and follow up with items:
```
[ protocolVersion, PoW, bloom, isLightNode, confirmationsEnabled, rateLimits ]
```

`protocolVersion`: version of the Whisper protocol
`PoW`: minimum PoW accepted by the peer
`bloom`: bloom filter of Whisper topic accepted by the peer
`isLightNode`: when true, the peer won't forward messages
`confirmationsEnabled`: when true, the peer will send message confirmations
`rateLimits`: is `[ RateLimitIP, RateLimitPeerID, RateLimitTopic ]` where each values is an integer with a number of accepted packets per second per IP, Peer ID, and Topic respectively

`bloom, isLightNode, confirmationsEnabled, and rateLimits` are all optional arguments in the handshake. However, if you specify optional field you MUST also specify all optional fields preceding it, in order to be unambiguous.

## Rate limiting

In order to provide an optional very basic Denial-of-Service attack protection, each node SHOULD define its own rate limits. The rate limits SHOULD be applied on IPs, peer IDs, and envelope topics.

Each node MAY decide to whitelist, i.e. do not rate limit, selected IPs or peer IDs.

If a peer exceeds node's rate limits, the connection between them MAY be dropped.

Each node SHOULD broadcast its rate limits to its peers using rate limits packet code (`0x14`). The rate limits is RLP-encoded information:

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

Keys management for PFS is described in [Perfect forward secrecy section](#perfect-forward-secrecy-pfs).

The Status protocols uses a few particular Whisper topics to achieve its goals.

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

Whisper is broadcast-based protocol. In theory, everyone could communicate using a single topic but that would be extremaly inefficient. Opposite would be using a unique topic for each conversation, however, this brings privacy concerns because it would be much easier to detect whether and when two parties have an active conversation.

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

If partitioned topic support is enabled by the Status client, it MUST listen to its paritioned topic. It MUST be generated using the algorithm above and active public key.

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

<!-- NOTE: commented out as it is no longer valid as of V1. - C.P. Oct 8, 2019
### Generic discovery topic

Generic discovery topic is a legacy topic used to handle all one-to-one chats. The newer implementation should rely on [Partitioned Topic](#partitioned-topic) and [Personal discovery topic](#personal-discovery-topic).

Generic discovery topic MUST be created following [Public chats](#public-chats) topic algorithm using string `contact-discovery` as a name. -->

### One-to-one topic

In order to receive one-to-one messages incoming from a public key `P`, the Status Client MUST listen to a [Contact Code Topic](#contact-code-topic) created for that public key.

### Group chat topic

Group chats does not have a dedicated topic. All group chat messages (including membership updates) are sent as one-to-one messages to multiple recipients.

## Message encryption

Even though, the protocol specifies an encryption layer that encrypts messages before passing them to the transport layer, Whisper protocol requires each Whisper message to be encrypted anyway.

Public and group messages are encrypted using symmetric encryption and the key is created from a channel name string. The implementation is available in [`shh_generateSymKeyFromPassword`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_generatesymkeyfrompassword) JSON-RPC method of go-ethereum Whisper implementation.

One-to-one messages are encrypted using asymmetric encryption.

## Message confirmations

Sending a message is a complex process where many things can go wrong. Message confirmations tell a node that a message originating from it has been received by its peers.

A node MAY send a message confirmation for any batch of messages received with a packet Messages Code (`0x01`).

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

The drawback of sending message confirmations is that it increases the noise in the network because for each sent message, a corresponding confirmation is broadcasted by one or more peers.

## Whisper V6 extensions

### Request historic messages

Sends a request for historic messages to a Mailserver. The Mailserver node MUST be a direct peer and MUST be marked as trusted (using `shh_markTrustedPeer`).

The request does not wait for the response. It merely sends a peer-to-peer message to the Mailserver and it's up to Mailserver to process it and start sending historic messages.

The drawback of this approach is that it is impossible to tell which historic messages are the result of which request.

It's recommended to return messages from newest to oldest. To move further back in time, use `cursor` and `limit`.

#### shhext_requestMessages

**Parameters**:
1. Object - The message request object:
   * `mailServerPeer` - `String`: Mailserver's enode address.
   * `from` - `Number` (optional): Lower bound of time range as unix timestamp, default is 24 hours back from now.
   * `to` - `Number` (optional): Upper bound of time range as unix timestamp, default is now.
   * `limit` - `Number` (optional): Limit the number of messages sent back, default is no limit.
   * `cursor` - `String` (optional): Used for paginated requests.
   * `topics` - `Array`: hex-encoded message topics.
   * `symKeyID` - `String`: an ID of a symmetric key to authenticate to Mailserver, derived from Mailserver password.

**Returns**:
`Boolean` - returns `true` if the request was sent.

The above `topics` is then converted into a bloom filter and then and sent to the Mailserver.

<!-- TODO: Clarify actual request with bloom filter to mailserver -->
