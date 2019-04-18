---
eip: 3
title: Status Secure Messaging Protocol
status: Draft
type: Standard
author: Adam Babik <adam@status.im>
created: 2019-04-18
updated: 2019-04-18
---

Status Secure And Decentralized Messaging Protocol
==================================================

- [Status Secure And Decentralized Messaging Protocol](#status-secure-and-decentralized-messaging-protocol)
- [Abstract](#abstract)
- [Terminology](#terminology)
- [Basic Assumption](#basic-assumption)
- [Protocol Overview](#protocol-overview)
- [Payload](#payload)
  - [Content types](#content-types)
  - [Message types](#message-types)
  - [Clock vs Timestamp and message ordering](#clock-vs-timestamp-and-message-ordering)
  - [Replies](#replies)
- [Whisper adapter](#whisper-adapter)
  - [Whisper node configuration](#whisper-node-configuration)
  - [Keys management](#keys-management)
  - [Encryption algorithms](#encryption-algorithms)
  - [Topic](#topic)
  - [Message encryption](#message-encryption)
- [Perfect Forward Secrecy (PFS)](#perfect-forward-secrecy-pfs)
- [Device syncing](#device-syncing)
- [One-to-one messages](#one-to-one-messages)
  - [Sending](#sending)
    - [Sending using PFS](#sending-using-pfs)
  - [Receiving](#receiving)
- [Public messages](#public-messages)
  - [Sending](#sending-1)
  - [Receiving](#receiving-1)
- [Group messages](#group-messages)
- [Offline messages](#offline-messages)
  - [Anonymity concerns](#anonymity-concerns)
- [Whisper V6 extensions (or Status Whisper Node)](#whisper-v6-extensions-or-status-whisper-node)
  - [New RPC methods](#new-rpc-methods)

# Abstract

Ethereum empowers users and developers to interact with totally new kind of applications called Dapps (Decentralized applications). These application allows to interact with the blockchain on a completely new level which is not only about exchanging values but also executing arbitrary logic. This logic can form very sophisticated programs like DAOs (Decentralized autonomous organizations). The missing part here is how users of Dapps can communicate securely and in a decentralized way with each other. Communication is an essential part of any activity. In this document, we specify a secure and decentralized messaging protocol that is capable of running on the Ethereum network.

# Terminology

* *Client*: a Whisper node implementing the protocol
* *Whisper node*: an Ethereum node with Whisper V6 enabled (in the case of geth, it's `--shh` option)
* *Status Whisper node*: an Ethereum node with Whisper V6 enabled and additional Whisper extensions described in [Whisper V6 extensions (or Status Whisper Node)](#whisper-v6-extensions-or-status-whisper-node)
* *Whisper network*: a group of Whisper nodes connected together through the internet connection and forming a graph
* *MailServer*: an Ethereum node with Whisper V6 enabled and a mail server registered capable of storing and providing offline messages
* *Message*: decrypted Whisper message
* *Envelope*: encrypted message with some metadata like topic and TTL sent between Whisper nodes; a symmetric or asymmetric key is needed to decrypt it and read the payload
* *Offline message*: an expired envelope stored by a Whisper node permanently

# Basic Assumption

This protocol assumes the following:
1. There MUST be an Ethereum node that is capable of discovering peers and implements Whisper V6 specification.
2. Participants of a given Whisper network in order to communicate with each other MUST accept messages with lowered PoW value. More in (Whisper node configuration)(#whisper-node-configuration).
3. Time MUST be synced between all nodes participating in the given network (this is intrinsic requirement of the Whisper specification as well). A clock drift between two peers larger than 20 seconds MAY result in discarding incoming messages.

# Protocol Overview

Notice: this protocol is documented post factum. The goal of it is to clearly present the current design and prepare the ground for its second version.

The implementation of this protocol is mainly done in https://github.com/status-im/status-react and https://github.com/status-im/status-go repositories.

The goal of this protocol is to allow people running Ethereum nodes with Whisper service enabled to exchange messages that are end-to-end encrypted in a way that guarantees [darkness to some extent](https://github.com/ethereum/go-ethereum/wiki/Achieving-Darkness).

It's important to notice that messages [are not limited to be text messages](#content-types) only. They can also have special meaning depending on the client implementation. For example, in the current implementation, there are message which informs about Eth requests.

This protocol consist of three components:
* payload
* Whisper adapter
* offline messaging.

[The payload section](#payload) describes how the messages are encoded and decoded and what each fields of a message means. This is required to properly interpret messages by the client.

Whisper adapter specifies interaction with the Whisper service with regards to keys management, configuration and attaching metadata required to properly forward and process messages.

Offline messaging describes how the protocol handles delivering messages when one or more participants are offline and the messages expire in the Whisper network.

The protocol does not specify additional things like peers discovery, running Whisper nodes, underlying p2p protocols etc.

# Payload

Payload is an array of bytes exchanged between peers. In the case of this protocol, the payload is encoded using [transit format](https://github.com/cognitect/transit-format). It is a text-based format so even encoded version is easily readable by humans.

Payload contains the following fields:
1. text `string`
2. content type `enum { text/plain }` (more in [Content types](#content-types))
3. message type `enum { public-group-user-message, user-message, group-user-message }` (more in [Message types](#message-types))
4. clock `int64`
5. timestamp `int64`
6. content `struct { chat-id string, text string }`

Example of a valid encoded payload:

```
["~#c4",["abc123","text/plain","~:public-group-user-message",154593077368201,1545930773682,["^ ","~:chat-id","testing-adamb","~:text","abc123"]]]
```

As you can see, the message is an array and each index value has its meaning:
* 0: `c4` is a decoder handler identification for the current payload format. Identifications allow to register handlers for many different types of payload
* 1: array which items correspond to the described payload fields above

For more details regarding serialization and deserialization please consult [transit format](https://github.com/cognitect/transit-format) specification.

## Content types

Content types are required for a proper interpretation of incoming messaages. Not each message is a plain text but may carry a different information.

The following content types MUST be supported:
* `text/plain` identifies a message which content is a plain text
* `sticker` TODO
* `status` TODO
* `command` TODO
* `command-request` TODO
* `emoji` TODO

TODO: select which are required and which are client-specific.

## Message types

Message types are required to decide how a particular message is encrypted (more in [Whisper > Message encryption](#message-encryption)) and what metadata needs to be attacjed (more in [Whisper > Topic](#topic)) when passing a message to the transport layer.

The following messages types MUST be supported:
* `public-group-user-message` is a message to the public group
* `user-message` is a private message
* `group-user-message` is a message to the private group.

## Clock vs Timestamp and message ordering

`timestamp` MUST be Unix time calculated when the message is created. Because the peers in the Whisper network should have synchronized time, `timestamp` values should be fairly accurate among all Whisper network participants.

`clock` SHOULD be calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, `clock`'s value is calculated based on the last received message in a particular chat: `last-message-clock-value + 1`. If there are no messages, `clock` is initialized with `timestamp`'s value.

`clock` value is used for the message ordering. Due to the used algorithm and distributed nature of the system, we achieve casual ordering which might produce counterintuitive results in some edge cases. For example, when one joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

## Replies

TBD

# Whisper adapter

Whisper in version 6 has been chosen as an messages exchange protocol because it was designed as an off-chain communication layer for the Ethereum nodes. It supports e2e encryption and uses epidemic spread to route data to all members of the network. It also provides [darkness to some extent](https://github.com/ethereum/go-ethereum/wiki/Achieving-Darkness).

However, using Whisper has a few tradeoffs:
* was not designed to handle huge number of messages
* was not designed to be real-time; some delays over a few seconods are expected
* does not scale well with the number of messages in the network

This protocol can operate using a Whisper service which requires this protocol implementation to run in the same process as well as Whisper's RPC API which might be provided by a separate Whisper node process via IPC or WebSocket.

There is some tight coupling between the payload and Whisper:
* Whisper message topic depends on the actual message type (see [Topic](#topic))
* Whisper message uses a different key (asymmetric or symmetric) depending on the actual message type (see [Keys management](#keys-management))

## Whisper node configuration

If you want to run a Whisper node and receive messages from Status clients, it must be properly cnofigured.

Whisper's Proof Of Work algorithm is used to deter denial of service and various spam/flood attacks against the Whisper network. The sender of a message must perform some work which in this case means processing time. Because Status' main client is a mobile client, this easily leads to battery draining and poor performance of the app itself. Hence, all clients MUST use the following Whisper node settings:
* proof-of-work not larger than `0.002`
* time-to-live not lower than `10` (in seconds)

TODO: provide an instruction how to start a Whisper node with proper configuration using geth.

## Keys management

The protocol requires a key (symmetric or asymmetric) for the following actions:
* signing a message (a private key)
* decrypting received messages (a private key or symmetric key).

As private keys and symmetric keys are required to process incoming messages, they must be available all the time and are stored in memory.

Keys management for PFS is described in [Perfect forward secrecy section](#perfect-forward-secrecy-pfs).

## Encryption algorithms

All encryption algorithms used by Whisper should be described in the [Whisper V6 specification](http://eips.ethereum.org/EIPS/eip-627).

Cryptographic algoritms used by PFS are described in [Perfect forward secrecy section](#perfect-forward-secrecy-pfs).

## Topic

There are two types of Whisper topics the protocol uses:
* static topic for `user-message` message type (also called _contact discovery topic_)
* dynamic topic based on a chat name for `public-group-user-message` message type.

The static topic is always the same and its hex representation is `0xf8946aac`. In fact, _the contact discovery topic_ is calculated using a dynamic topic algorithm described below with a constant name `contact-discovery`.

Having only one topic for all private chats has an advantage as it's very hard to figure out who talks to who. A drawback is that everyone receives everyone's messages but they can decrypt only these they have private keys for.

A dynamic topic is derived from a string using the following algorithm:

```
var hash []byte

hash = keccak256(name)

# Whisper V6 specific
var topic [4]byte

topic_len = 4

if len(hash) < topic_len {
    topic_len = len(hash)
}

for i = 0; i < topic_len; i++ {
    topic[i] = hash[i]
}
```

## Message encryption

The protocol distinguishes messages encrypted using asymmetric and symmetric encryption.

Symmetric keys are created using [`shh_generateSymKeyFromPassword`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_generatesymkeyfrompassword) Whisper V6 RPC API method which accepts one param, a string.

Messages encrypted with asymmetric encryption should be encrypted using recipient's public key so that only the recipient can decrypt them.

Encryption of messages supporting PFS is described in [Perfect Forward Secrecy](#perfect-forward-secrecy-pfs) section.

# Perfect Forward Secrecy (PFS)

Additionally to encrypting messages on the Whisper level, the protocol supports PFS specification.

A message payload is first encrypted following the PFS specification and then it is encrypted once again following the Whisper specification and this protocol.

As not all messages are encrypted with PFS, a following strategy MAY be used:
1. First, message is decrypted on the Whisper level
2. Try to decrypt the message payload using PFS algorithm
2.1. If successful, pass the decrypted value to (3)
2.2. If failed, pass the unchanged payload to (3)
3. Decode the payload as described in [Paylooad](#payload) section

TODO: link to a separate document (currently in the PR).

[PFS in Status.im docs](https://status.im/research/pfs.html)

# Device syncing

TODO: link to a separate document.

# One-to-one messages

One-to-one messages are also known as private messages. These are the messages sent beween two participants of the conversation.

## Sending

Sending a message is fairly easy and relies on the Whisper RPC API, however, some preparation is needed:

1. Obtain a public key of the recipient of the message
2. Add your private key to Whisper using [`shh_addPrivateKey`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_addprivatekey) and save the result as `sigKeyID`
3. Call [`shh_post`(https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_post) with the following settings:
   1. `pubKey` MUST be a hex-encoded public key of the message recipient
   2. `sig` MUST be set to `sigKeyID`
   3. `ttl` MUST be at least `10` (it is in seconds)
   4. `topic` MUST be set accordingly to [Topic](#topic) section and hex-encoded
   5. `payload` MUST be a hex-encoded string
   6. `powTime` MAY be arbitrary but should be enough to perform proof-of-work
   7. `powTarget` MUST be equal or lower than `0.002`.

Note: these instructions are for the Whisper V6 RPC API. If you use Whisper service directly or Go `shhclient`, the parameters might have different types.

Learn more following [Whisper V6 RPC API](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API).

### Sending using PFS

When one decides to use PFS, the flow is the same but the payload MUST be additionally encrypted following the [PFS specification](#pfs) before being hex-endoded and passed to `shh_post`.

## Receiving

Receiving private messages depends on Whisper filters idea. Upon receiving, messages are first matched by a topic and then trying to be decrypted using user's private key.

1. Add your private key to Whisper using [`shh_addPrivateKey`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_addprivatekey) and save the result as `sigKeyID`
2. Call [`shh_subscribe`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_subscribe) with criteria:
   1. `minPow` MUST be at least `0.002`
   2. `topics` MUST be list of hex-encoded topics you expect messages to receive from (follow [Topic](#topic) section)
   3. `allowP2P` MUST be set to `true` if offline messages are supported, otherwise can be `false`.

Alternative method is to use [`shh_newMessageFilter`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_newmessagefilter) which takes the same criteria object and then periodically calling [`shh_getFilterMessages`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_getfiltermessages) method.

Learn more following [Whisper V6 RPC API](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API).

# Public messages

Public messages are encrypted with a symmetric key which is publicly known so anyone can participate in the conversation.

The fact that anyone can participate makes the public chats voulnerable to spam attacks. Also, there are no moderators of these chats.

## Sending

1. Calculate a symmetric key using [`shh_generateSymKeyFromPassword`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_generatesymkeyfrompassword) passing a public chat name as a string and save the result to `symKeyID`
2. Call [`shh_post`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_post) with the following settings:
   1. `symKeyID` MUST be set to `symKeyID`
   2. `sig` MUST be set to `sigKeyID`
   3. `ttl` MUST be at least `10` (it is in seconds)
   4. `topic` MUST be set accordingly to [Topic](#topic) section and hex-encoded,
   5. `payload` MUST be a hex-encoded string,
   6. `powTime` MAY be arbitrary but should be enough to perform proof-of-work
   7. `powTarget` MUST be equal or lower than `0.002`.

Learn more following [Whisper V6 RPC API](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API).

## Receiving

Receiving public messages depends on Whisper filters idea. Upon receiving, messages are first matched by a topic and then trying to be decrypted using a symmetric key.

1. Calculate a symmetric key using [`shh_generateSymKeyFromPassword`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_generatesymkeyfrompassword) passing public chat name as a string and save the result to `symKeyID`
2. Call [`shh_subscribe`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_subscribe) with criteria:
   1. `minPow` MUST be at least `0.002`
   2. `topics` MUST be list of hex-encoded topics you expect messages to receive from (follow [Topic](#topic) section)
   3. `allowP2P` MUST be set to `true` if offline messages are supported, otherwise can be `false`.

Alternative method is to use [`shh_newMessageFilter`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_newmessagefilter) which takes the same criteria object and then periodically calling [`shh_getFilterMessages`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_getfiltermessages) method.

Learn more following [Whisper V6 RPC API](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API).

# Group messages

TODO: describe how to send a group message starting from adding a key in Whisper etc.

# Offline messages

In the case of mobile clients which are often offline, there is a strong need to have an ability to download offline messages. By offline messages, we mean messages sent into the Whisper network and expired before being collected by the recipient. A message stays in the Whisper network for a duration specified as `TTL` (time-to-live) property.

A Whisper client needs to register a mail server instance which will be used by [geth's Whisper service](https://github.com/ethereum/go-ethereum/blob/v1.8.23/whisper/whisperv6/whisper.go#L209-L213).

`MailServer` is an interface with two methods:
* `Archive(env *Envelope)`
* `DeliverMail(whisperPeer *Peer, request *Envelope)`

If a mail server is registered for a given Whisper client, it will save all incoming messages on a local disk (this is the simplest implementation, it can store the messages wherever it wants, also using technologies like swarm and IPFS) in the background.

Notice that each node is meant to be independent and SHOULD keep a copy of all historic messages. High Availability (HA) can be achieved by having multiple nodes in different locations. Additionally, each node is free to store messages in a way which provides storage HA as well.

Saved messages are delivered to a requester (another Whisper peer) asynchronously as a response to `p2pMessageCode` message code. This is not exposed as a JSON-RPC method in `shh` namespace but it's exposed in status-go as `shhext_requestMessages` and blocking `shh_requestMessagesSync`. Read more about [Whisper V6 extensions](#whisper-v6-extensions-or-status-whisper-node).

In order to receive historic messages from a filter, p2p messages MUST be allowed when creating the filter. Receiving p2p messages is implemented in [geth's Whisper V6 implementation](https://github.com/ethereum/go-ethereum/blob/v1.8.23/whisper/whisperv6/whisper.go#L739-L751).

## Anonymity concerns

In order to use a mail server, a given node needs to connect to it directly, i.e. add the mail server as its peer and mark it as trusted. This means that the mail server is able to send direct p2p messages to the node instead of broadcasting them. Effectively, it knows which topics the node is interested in, when it is online as well as many metadata like IP address.

# Whisper V6 extensions (or Status Whisper Node)

Protocol's target is to be compliant with [the Whisper V6 Specification](https://github.com/ethereum/go-ethereum/wiki/Whisper). It should not matter which implementation is used as long as the implementation follow the Whisper V6 Specification.

However, we added a few extensions, message codes and RPC methods to the Whisper V6 service in order to provide better user experience or due to efficiency requirements.

All described addons are implemented in [status-im/whisper fork](https://github.com/status-im/whisper).

## New RPC methods

TODO: provide a list of RPC methods from `shhext` API which are relevant to this spec.
