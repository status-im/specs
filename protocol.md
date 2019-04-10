Status Secure And Decentralized Messaging Protocol
==================================================

- [Status Secure And Decentralized Messaging Protocol](#status-secure-and-decentralized-messaging-protocol)
- [Abstract](#abstract)
- [Introduction](#introduction)
- [Terminology](#terminology)
- [Basic Assumption](#basic-assumption)
- [Protocol Overview](#protocol-overview)
- [Payload](#payload)
	- [Content types](#content-types)
	- [Message types](#message-types)
	- [Clock vs Timestamp And Message Ordering](#clock-vs-timestamp-and-message-ordering)
	- [Quoting](#quoting)
- [Whisper adapter](#whisper-adapter)
	- [Whisper node configuration](#whisper-node-configuration)
	- [Keys management](#keys-management)
	- [Encryption algorithms](#encryption-algorithms)
	- [Topic](#topic)
	- [Encryption](#encryption)
- [Perfect Forward Secrecy (PFS)](#perfect-forward-secrecy-pfs)
- [Device syncing](#device-syncing)
- [One-to-one messages](#one-to-one-messages)
- [Public messages](#public-messages)
- [Group messages](#group-messages)
- [Offline messages](#offline-messages)
	- [Anonymity concerns](#anonymity-concerns)
- [Whisper V6 extensions (or Status Whisper Node)](#whisper-v6-extensions-or-status-whisper-node)

# Abstract

Ethereum empowers users and developers to interact with totally new kind of applications called Dapps (Decentralized applications). These application allows to interact with the blockchain on a completely new level which is not only about exchanging values but also executing arbitrary logic. This logic can form very sophisticated programs like DAOs (Decentralized autonomous organizations). The missing part here is how users of Dapps can communicate securely and in a decentralized way with each other. Communication is an essential part of any activity. In this document, we specify a secure and decentralized messaging protocol that is capable of running on the Ethereum network.

# Introduction

TBD

# Terminology

* *Client*: a Whisper node implementing the protocol
* *Whisper node*: an Ethereum node with Whisper V6 enabled (in the case of geth, it's `--shh` option)
* *Status Whisper node*: an Ethereum node with Whisper V6 enabled and additional Whisper extensions described in [Whisper V6 extensions (or Status Whisper Node)](#whisper-v6-extensions-or-status-whisper-node)
* *MailServer*: an Ethereum node with Whisper V6 enabled and a mail server registered capable of storing and providing offline messages
* *Message*: decrypted Whisper message
* *Envelope*: encrypted message with some metadata like topic and TTL echanged between Whisper nodes; a symmetric or assymetric key is needed to decrypt it and read the payload
* *Offline message*: an expired envelope stored by a Whisper node permanently

# Basic Assumption

This protocol assumes the following:
1. There is an Ethereum node that is capable of discovering peers and implements Whisper V6 service.
2. Participants of a given Whisper network in order to communicate with each other accept messages with lowered PoW value.
3. Time is synced between all nodes participaiting in the given network (this is intrinsic Whisper requirement).

# Protocol Overview

Notice: this protocol is documented post factum. The goal of it is to clearly present the current design and prepare the ground for its second version.

The implementation of this protocol is mainly done in https://github.com/status-im/status-react and https://github.com/status-im/status-go repositories.

The goal of this protocol is to allow people running Ethereum nodes with Whisper service enabled to exchange messages that are encrypted end-to-end in a way that guarantees darkness to some extent (messages are broadcasted through the network rather than sent directly or using a known path).

It's important to notice that messages are not limited to be text messages only. They can also have special meaning depending on the client implementation. For example, in the current implementation, there are message which informs about Eth requests or current user's location.

This protocol consist of three components:
* payload specification
* Whisper adapter
* Offline messaging.

The payload specification describes how the messages are encoded and decoded and what each fields mean. This is required to properly interpret messages by the client.

Whisper adapter specifies interaction with the Whiper service with regards to keys management, configuration and some metadata (like topic) required to properly process and encrypt/decrypt messages.

Offline messaging describes how the protocol handles delivering messages when one or more participants were offline and the messages expired in the network.

The protocol does not specify things like peers discovery, however, some notes and the current implementation will be described in an appendix.

# Payload

Payload is encrypted and decrypted using [transit format](https://github.com/cognitect/transit-format). It is a text-based format so that each decrypted message's payload can be easily read.

Payload contains the following fields:
* text `string`
* content type `enum { text/plain }`
* message type `enum { public-group-user-message, user-message }`
* clock `int64`
* timestamp `int64`
* content `struct { chat-id string, text string }`

Example of a valid message:

```
["~#c4",["abc123","text/plain","~:public-group-user-message",154593077368201,1545930773682,["^ ","~:chat-id","testing-adamb","~:text","abc123"]]]
```

As you can see, the message is an array and each index value has its meaning:
* 0: `c4` is a decoder handler identificator for the current payload format. Identificators allow to register handler for many different types of messages
* 1: array which items correspond to the described payload fields above

For more details regarding serialization and deserialization please consult [transit format](https://github.com/cognitect/transit-format) specification.

## Content types

Here is a list of supported content types:
* `text/plain` identifies a message which content is a plain text

TODO: are there other types? How is it useful? Sending Eth requests?

## Message types

Here is a list of supported message types:
* `public-group-user-message` is a message to the public group
* `user-message` is a private message.

TODO: how is it useful? What are other supported message types?

## Clock vs Timestamp And Message Ordering

`timestamp` is Unix time calculated when the message is created (TODO: does it come from Whisper or the system?)

`clock` is calculated using the algorithm of [Lamport timestamps](https://en.wikipedia.org/wiki/Lamport_timestamps). When there are messages available in a chat, `clock`'s value is calculated as `last-message-clock-value + 1`. If there are no messages, `clock` is initialized with `timestamp`'s value.

`clock` value is used for the message ordering. Due to the used algorithm, distributed nature of the system and physics, we achieve casual ordering which might produce counterintuitive results in some edge cases. For example, when one joins a public chat and sends a message before receiving the exist messages, their message `clock` value might be lower and the message will end up in the past when the historical messages are fetched.

## Quoting

TBD

# Whisper adapter

Whisper has been chosen as an messages exchange protocol because it was designed as an off-chain communcation layer for the Ethereum nodes. It supports e2e encryption and uses epidemic spread to route data to all members of the network. It also provides [darkness to some extent](https://github.com/ethereum/go-ethereum/wiki/Achieving-Darkness).

However, Whisper was not designed to handle huge number of messages and real-time communication. These are the tradeoffs that we accepted when implementing this protocol.

This protocol can either work using a Whisper service which requires a protocol implementation to run in the same process as a Whisper node as well as using a Whisper client which might run as a separate Ethereum node and communicate through IPC or WebSocket.

There is some tight coupling between the payload and Whisper:
* Whisper message topic depends on the actual message type (see [Topic](#topic))
* Whisper message uses a different key (asymmetric or symmetric) depending on the actual message type (see [Keys management](#keys-management))

## Whisper node configuration

If you want to run a Whisper node and receive messages from Status clients, you need to configure it properly, otherwise, some message might not be received.

Whisper's Proof Of Work algorithm is used to to deter denial of service and various spam/flood attacks against the Whisper network. The sender of a message must perform some work which in this case means performing some computations. Because Status main client is a mobile client, this easily leads to battery draining and poor performance of the app itself. Hence, these clients send envelopes with the following settings:
* proof-of-work equal `0.002`
* proof-of-work time equal to `10`
* time-to-live equal `10`

If you want to receive messages from a mobile client, you need to run Whisper node with PoW set to `0.002`. In case of `geth`, this option can be overriden with `-shh.pow=0.002` flag.

TODO: provide an instruction how to start a Whisper node with proper configuration using geth.

## Keys management

TBD

## Encryption algorithms

TBD (link to Whisper spec)

## Topic

There are two types of Whisper topics the protocol uses:
* static topic for private chats (also called _contact discovery topic_)
* dynamic topic based on a chat name for public chats

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

## Encryption

Protocol distinguishes messages encrypted using asymmetric and symmetric encryptions.

Symmetric keys are created using `shh_generateSymKeyFromPassword` Whisper JSON-RPC method which accepts one param, a string.

Messages encrypted with asymmetric encryption should be encrypted using recipeint's public key so that only the recipient can decrypt them.

# Perfect Forward Secrecy (PFS)

TODO: link to a separate document (currently in the PR).

[PFS in Status.im docs](https://status.im/research/pfs.html)

# Device syncing

TODO: link to a separate document.

# One-to-one messages

One-to-one messages are also known as private messages.

TODO: describe how to send a 1-1 message starting from adding a key in Whisper etc.

# Public messages

TODO: describe how to send a public message starting from adding a key in Whisper etc.TBD

# Group messages

TODO: describe how to send a group message starting from adding a key in Whisper etc.

# Offline messages

In the case of mobile clients which are often offline, there is a strong need to have an ability to download offline messages. By offline messages, we mean messages sent into the Whisper network and expired before being collected by the client. A message stays in the Whisper network for a duration specified as TTL (time-to-live) property.

Whisper client needs to register a mail server instance which will be used by [geth's Whisper service](https://github.com/ethereum/go-ethereum/blob/v1.8.23/whisper/whisperv6/whisper.go#L209-L213).

`MailServer` is an interface with two methods:
* `Archive(env *Envelope)`
* `DeliverMail(whisperPeer *Peer, request *Envelope)`

Archiving happens in the background automatically. If a mail server is registered for a given Whisper client, it will save all incoming messages on a local disk (this is the simplest implementation, it can store the messages wherever it wants, also using technologies like swarm and IPFS).

Notice that each node is meant to be independent and keeps a copy of all historic messages. High Availability (HA) can be achieved by having multiple nodes in different locations. Additionally, each node is free to store messages in a way which provides storage HA as well.

Saved messages are delivered to a requester (another Whisper peer) asynchronously as a response to `p2pMessageCode` message code. This is not exposed as a JSON-RPC method in `shh` namespace but it's exposed in status-go as `shhext_requestMessages` and blocking `shh_requestMessagesSync`.

In order to receive historic messages from a filter, p2p messages must be allowed when creating the filter. Receiving p2p messages is implemented in [geth's Whisper V6 implementation](https://github.com/ethereum/go-ethereum/blob/v1.8.23/whisper/whisperv6/whisper.go#L739-L751).

## Anonymity concerns

In order to use a mail server, a given node needs to connect to it directly, i.e. add the mail server as its peer and mark it as trusted. This means that the mail server is able to send direct p2p messages to the node instead of broadcasing them. Effectively, it knows which topics the node is interested in, when it is online as well as many metadata like IP address.

# Whisper V6 extensions (or Status Whisper Node)

TBD
