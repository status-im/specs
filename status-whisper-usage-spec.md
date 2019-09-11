# Status Whisper Usage Specification

> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

- [Status Whisper Usage Specification](#status-whisper-usage-specification)
  - [Abstract](#abstract)
  - [Reason](#reason)
  - [Terminology](#terminology)
  - [Whisper node configuration](#whisper-node-configuration)
  - [Topics](#topics)
    - [Contact code topic](#contact-code-topic)
    - [Partitioned topic](#partitioned-topic)
    - [Public chats](#public-chats)
    - [Personal discovery topic](#personal-discovery-topic)
    - [Generic discovery topic](#generic-discovery-topic)
    - [One-to-one topic](#one-to-one-topic)
    - [Group chat topic](#group-chat-topic)
  - [Message encryption](#message-encryption)
  - [Whisper V6 extensions (or Status Whisper Node)](#whisper-v6-extensions-or-status-whisper-node)

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

Whisper's Proof Of Work algorithm is used to deter denial of service and various spam/flood attacks against the Whisper network. The sender of a message MUST perform some work which in this case means processing time. Because Status' main client is a mobile client, this easily leads to battery draining and poor performance of the app itself. Hence, all clients MUST use the following Whisper node settings:
* proof-of-work not larger than `0.002`
* time-to-live not lower than `10` (in seconds)

## Topics

The Status protocols uses a few particular Whisper topics to achieve its goals.

### Contact code topic

Contact code topic is used for ???

<!-- TODO: describe who should listen to it -->

It MUST be created following the algorithm below:
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

Each Status Client SHOULD listen to this topic in order to receive ???

### Generic discovery topic

Generic discovery topic is a legacy topic used to handle all one-to-one chats. The newer implementation should rely on [Partitioned Topic](#partitioned-topic) and [Personal discovery topic](#personal-discovery-topic).

Generic discovery topic MUST be created following [Public chats](#public-chats) topic algorithm using string `contact-discovery` as a name.

### One-to-one topic

In order to receive one-to-one messages incoming from a public key `P`, the Status Client MUST listen to a [Contact Code Topic](#contact-code-topic) created for that public key.

### Group chat topic

TODO

## Message encryption

Even though, the protocol specifies an encryption layer that encrypts messages before passing them to the transport layer, Whisper protocol requires each Whisper message to be encrypted anyway.

Public and group messages are encrypted using symmetric encryption and the key is created from a channel name string. The implementation is available in [`shh_generateSymKeyFromPassword`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_generatesymkeyfrompassword) JSON-RPC method of go-ethereum Whisper implementation.

One-to-one messages are encrypted using asymmetric encryption.

## Whisper V6 extensions (or Status Whisper Node)

TOODO

<!--TODO: provide a list of required JSON-RPC methods, if any -->
