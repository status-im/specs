# Status Whisper Usage Specification

> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

- [Status Whisper Usage Specification](#status-whisper-usage-specification)
  - [Abstract](#abstract)
  - [Reason](#reason)
  - [Terminology](#terminology)
  - [Whisper node configuration](#whisper-node-configuration)
  - [Keys management](#keys-management)
    - [Contact code topic](#contact-code-topic)
    - [Partitioned topic](#partitioned-topic)
    - [Public chats](#public-chats)
    - [Personal discovery topic](#personal-discovery-topic)
    - [Generic discovery topic](#generic-discovery-topic)
    - [One-to-one topic](#one-to-one-topic)
    - [Group chat topic](#group-chat-topic)
  - [Message encryption](#message-encryption)
  - [Whisper V6 extensions](#whisper-v6-extensions)
    - [Request historic messages](#request-historic-messages)
      - [shhext_requestMessages](#shhext_requestmessages)
    <!-- - [Personal discovery topic](#personal-discovery-topic) C.P. Oct 8, 2019 --> 
    <!-- - [Generic discovery topic](#generic-discovery-topic) C.P. Oc8, 2019 -->
    - [One-to-one topic](#one-to-one-topic)
    - [Group chat topic](#group-chat-topic)
  - [Message encryption](#message-encryption)
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

## Whisper node configuration

If you want to run a Whisper node and receive messages from Status clients, it must be properly configured.

Whisper's Proof Of Work algorithm is used to deter denial of service and various spam/flood attacks against the Whisper network. The sender of a message must perform some work which in this case means processing time. Because Status' main client is a mobile client, this easily leads to battery draining and poor performance of the app itself. Hence, all clients MUST use the following Whisper node settings:
* proof-of-work requirement not larger than `0.002`
* time-to-live not lower than `10` (in seconds)

<!-- TODO: provide an instruction how to start a Whisper node with proper configuration using geth.-->

<!-- @TODO: is there a higher bound -->

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

## Whisper V6 extensions

### Request historic messages

Sends a request for historic messages to a Mailserver. The Mailserver node MUST be a direct peer and MUST be marked a trusted (using `shh_markTrustedPeer`).

The request does not wait for the response. It marely sends a peer-to-peer message to the Mailserver and it's up to Mailserver to process it and start sending historic messages.

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
