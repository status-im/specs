Status Secure And Decentralized Messaging Protocol
==================================================

# Abstract

Ethereum empowers users and developers to interact with totally new kind of applications called Dapps (Decentralized applications). These application allows to interact with the blockchain on a completely new level which is not only about exchanging values but also executing arbitrary logic. This logic can form very sophisticated programs like DAOs (Decentralized autonomous organizations). The missing part here is how users of Dapps can communicate securely and in a decentralized way with each other. Communication is an essential part of any activity. In this document, we specify a secure and decentralized messaging protocol that is capable of running on the Ethereum network.

# Introduction

TBD

# Terminology

* *Client*: a Whisper node implementing the protocol
* *Whisper node*: an Ethereum node with Whisper V6 enabled (in the case of geth, it's `--shh` option)
* *MailServer*: an Ethereum node with Whisper V6 enabled and a mail server registered capable of storing and providing offline messages
* *Message*: decrypted Whisper message
* *Envelope*: encrypted message with some metadata like topic and TTL echanged between Whisper nodes; a symmetric or assymetric key is needed to decrypt it and read the payload
* *Offline message*: an expired envelope stored by a Whisper node permanently

# Basic Assumption

This protocol assumes that there is an Ethereum node that is capable of discoverying peers and implements Whisper V6 service. It also assumes that the participants of a given Whisper network accept messages with lowered PoW value.

# Protocol Overview

Notice: this protocol is documented post factum. The goal of it is to clearly present the current design and prepare the ground for its second version.

The implementation of this protocol is mainly done in https://github.com/status-im/status-react and https://github.com/status-im/status-go repositories.

The goal of this protocol is to allow people running Ethereum nodes with Whisper service enabled to exchange messages that are encrypted end-to-end in a way that guarantees darkness to some extent (messages are broadcasted through the network rather than send directly or using a known path).

It's important to notice that messages are not limited to be text messages only. They can also have special meaning depending on the client implementation. For example, in the current implementation, there are message which informs about Eth requests or current user's location.

This protocol consist of two components:
* payload specification
* Whisper adapter.

The payload specification describes how the messages are encoded and decoded and what each fields mean. This is required to properly interpret messages by the client.

Whisper adapter specifies interaction with the Whiper service with regards to keys management, configuration and some metadata (like topic) required to properly process and encrypt/decrypt messages.

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

## Message ordering

TBD

## Quoting

TBD

# Whisper adapter

Whisper has been chosen as an messages exchange protocol because it was designed as an off-chain communcation layer for the Ethereum nodes. It supports e2e encryption and uses epidemic spread to route data to all members of the network. It also provides [darkness to some extent](https://github.com/ethereum/go-ethereum/wiki/Achieving-Darkness).

However, Whisper was not designed to handle huge number of messages and real-time communication. These are the tradeoffs that we accepted when implementing this protocol.

This protocol can either work using a Whisper service which requires a protocol implementation to run in the same process as a Whisper node as well as using a Whisper client which might run as a separate Ethereum node and communicate through IPC or WebSocket.

There is some tight coupling between the payload and Whisper:
* Whisper message topic depends on the actual message type
* Whisper message uses a different key (asymmetric or symmetric) depending on the actual message type

## Whisper node configuration

If you want to run a Whisper node and receive messages from Status clients, you need to configure it properly, otherwise, some message might not be received.

Whisper's Proof Of Work algorithm is used to to deter denial of service and various spam/flood attacks against the Whisper network. The sender of a message must perform some work which in this case means performing some computations. Because Status main client is a mobile client, this easily leads to battery draining and poor performance of the app itself. Hence, these clients send envelopes with the following settings:
* proof-of-work equal `0.002`
* proof-of-work time equal to `10`
* time-to-live equal `10`

If you want to receive messages from a mobile client, you need to run Whisper node with PoW set to `0.002`. In case of `geth`, this option can be overriden with `-shh.pow=0.002` flag.

## Keys management

TBD

## Topic

There are two types of Whisper topics the protocol uses:
* static topic for private chats
* dynamic topic based on a chat name for public chats

Static topic is always the same and its hex representation is `0xf8946aac`. It is also called a discovery topic. Having only one topic for all private chats has an advantage as it's very hard to figure out who talks to who. A drawback is that everyone receives everyone's messages but they can decrypt only these they have private keys for.

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

TBD

# Device syncing

TBD

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

Archiving happens in the background automatically. If a mail server is registered for a given Whisper client, it will save all incoming messages on a local disk (this is the simplest implementation, it can store the messages wherever it wants).

Saved messages are delivered to a requester (another Whisper peer) asynchronously as a response to `p2pMessageCode` message code. This is not exposed as a JSON-RPC method in `shh` namespace but it's exposed in status-go as `shhext_requestMessages` and blocking `shh_requestMessagesSync`.

In order to receive historic messages from a filter, p2p messages must be allowed when creating the filter. Receiving p2p messages is implemented in [geth's Whisper V6 implementation](https://github.com/ethereum/go-ethereum/blob/v1.8.23/whisper/whisperv6/whisper.go#L739-L751).

# Whisper V6 extensions

TBD
