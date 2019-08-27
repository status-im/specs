# Status Client Specification

> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Dean Eigenmann <dean@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Table of Contents

- [Abstract](#abstract)
- [P2P Overlay](#p2p-overlay)
- [Node discovery and roles](#node-discovery-and-roles)
    - [Bootstrapping](#bootstrapping)
    - [Discovery](#discovery)
    - [Mailservers](#mailservers)
    - [Mobile nodes](#mobile-nodes)
- [Whisper adapter](#whisper-adapter)
    - [Node configuration](#node-configuration)
    - [Keys management and encryption](#keys-management-and-encryption)
    - [Topics](#topics)
- [Secure Transport](#secure-transport)
- [Data Sync](#data-sync)
- [Payloads and clients](#payloads-and-clients)
- [Design Rationale](#design-rationale)
    - [P2P Overlay](#p2p-overlay-1)
        - [Why devp2p? Why not use libp2p?](#why-devp2p-why-not-use-libp2p)
        - [What about other RLPx subprotocols like LES, and Swarm?](#what-about-other-rlpx-subprotocols-like-les-and-swarm)
        - [Why do you use Whisper?](#why-do-you-use-whisper)
        - [I heard you were moving away from Whisper?](#i-heard-you-were-moving-away-from-whisper)
        - [Why is PoW for Whisper set so low?](#why-is-pow-for-whisper-set-so-low)
        - [Why do you not use Discovery v5 for node discovery?](#why-do-you-not-use-discovery-v5-for-node-discovery)
- [Footnotes](#footnotes)
- [Acknowledgements](#acknowledgements)

## Abstract

In this specification, we describe how to write a Status client for
communicating with other Status clients.

We present a reference implementation of the protocol <sup>1</sup> that is used
in a command line client <sup>2</sup> and a mobile app <sup>3</sup>.

This document consists of two parts. The first outlines the specifications that
have to be implemented in order to be a full Status client. The second gives a design rationale and answers some common questions.

## Protocol layers

Implementing a Status clients means implementing the following layers. Additionally, there are separate specifications for things like key management and account lifecycle.

| Layer             | Purpose                         | Technology                   |
|-------------------|---------------------------------|------------------------------|
| Data and payloads | End user functionality          | 1:1, group chat, public chat |
| Data sync         | Data consistency                | MVDS Ratchet                 |
| Secure transport  | Confidentiality, PFS, etc       | Double Ratchet               |
| Transport privacy | Routing, Metadata protection    | Whisper                      |
| P2P Overlay       | Overlay routing, NAT traversal  | devp2p                       |

## P2P Overlay

Status clients run on the public Ethereum network, as specified by the devP2P
network protocols. devP2P provides a protocol for node discovery which is in
draft mode
[here](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md). See
more on node discovery and management in the next section.

To communicate between Ethereum nodes, the [RLPx Transport
Protocol, v5](https://github.com/ethereum/devp2p/blob/master/rlpx.md) is used, which
allows for TCP-based communication between nodes.

On top of this we run the RLPx-based subprotocol [Whisper
v6](https://eips.ethereum.org/EIPS/eip-627) for privacy-preserving messaging.

## Node discovery and roles

There are four types of node roles:
1. Bootstrap nodes
2. Whisper relayers
3. Mailservers (servers and clients)
4. Mobile nodes (Status Clients)

To implement a standard Status client you MUST implement the last node type. The
other node types are optional, but we RECOMMEND you implement a mailserver
client mode, otherwise the user experience is likely to be poor.

### Bootstrapping

To connect to other Status nodes you need to connect to a bootstrap node. These
nodes allow you to discover other nodes of the network.

Currently the main bootstrap nodes are provided by Status Gmbh, but anyone can
run these provided they are connected to the rest of the Whisper network.

<!-- TODO: Add a link to bootstrap nodes -->

### Discovery

To implement a Status client you need to discover peers to connect to. We use a
light discovery mechanism based on a combination of [Discovery
v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) and
[Rendezvous Protocol](https://github.com/libp2p/specs/tree/master/rendezvous),
(with some
[modifications](https://github.com/status-im/rendezvous#differences-with-original-rendezvous)).
Additionally, some static nodes MAY also be used.

This part of the system is currently underspecified and requires further detail.

<!-- TODO: This section is way too vague, amend with concrete spec for how to do discovery of Status nodes. -->
<!-- TODO: Add info on peer list, Discover v5 timespan, static nodes etc - should be enough to discover nodes from a different stack -->

### Mailservers

Whisper Mailservers are special nodes that helps with offline inboxing. This
means you can receive Whisper envelopes after they have expired, which is useful
if they are sent while a node is offline. They operate on a store-and-forward
model.

<!-- TODO: Add a link to mailserver spec spec -->

### Mobile nodes

This is a Whisper node which connects to part of the Whisper network. It MAY
relay messages. See next section for more details on how to use Whisper to
communicate with other Status nodes.

## Whisper adapter

Once a Whisper node is up and running there are some specific settings required
to commmunicate with other Status nodes.

### Node configuration

Whisper's Proof Of Work algorithm is used to deter denial of service and various
spam/flood attacks against the Whisper network. The sender of a message must
perform some work which in this case means processing time. Because Status' main
client is a mobile client, this easily leads to battery draining and poor
erformance of the app itself. Hence, all clients MUST use the following Whisper
node settings:

* proof-of-work not larger than `0.002`
* time-to-live not lower than `10` (in seconds) <!-- @TODO: is there a higher bound -->

<!--
TODO: Decide whether to include this or not

There is some tight coupling between the payload and Whisper:
* Whisper message topic depends on the actual message type (see [Topic](#topic))
* Whisper message uses a different key (asymmetric or symmetric) depending on the actual message type (see [Keys management](#keys-management))
-->

### Keys management and encryption

The protocol requires a key (symmetric or asymmetric) for the following actions:
* signing a message (a private key)
* decrypting received messages (a private key or symmetric key).

As private keys and symmetric keys are required to process incoming messages,
they must be available all the time and are stored in memory.

All encryption algorithms used by Whisper are described in the [Whisper V6
specification](http://eips.ethereum.org/EIPS/eip-627).

Symmetric keys are created using
[`shh_generateSymKeyFromPassword`](https://github.com/ethereum/go-ethereum/wiki/Whisper-v6-RPC-API#shh_generatesymkeyfrompassword)
Whisper V6 RPC API method which accepts one param, a string.

Messages encrypted with asymmetric encryption should be encrypted using
recipient's public key so that only the recipient can decrypt them.

Key management and encryption for conversational security is described in [Conversational
Security](#conversational-security), which provides properties such as forward secrecy.

<!-- TODO: Outlink to conversational security spec -->

### Topics

There are two types of Whisper topics the protocol uses:
* static topic for `user-message` message type (also called _contact discovery topic_)
* dynamic topic based on a chat name for `public-group-user-message` message type.

The static topic is always the same and its hex representation is `0xf8946aac`.
In fact, _the contact discovery topic_ is calculated using a dynamic topic
algorithm described below with a constant name `contact-discovery`.

<!-- TODO: Update this, this looks different with partitioned topic -->
Having only one topic for all private chats has an advantage as it's very hard
to figure out who talks to who. A drawback is that everyone receives everyone's
messages but they can decrypt only these they have private keys for.

A dynamic topic is derived from a string using the following algorithm:

```golang
var hash []byte

hash = keccak256(name)

// Whisper V6 specific
var topic [4]byte

topic_len = 4

if len(hash) < topic_len {
    topic_len = len(hash)
}

for i = 0; i < topic_len; i++ {
    topic[i] = hash[i]
}
```

### Whisper v6 extensions

Outside of Whisper v6, there are some extensions, message codes and RPC methods that MAY be useful for client implementers. An implementation of this can be found in a fork of Whisper [here](https://github.com/status-im/whisper).

<!--TODO: provide a list of RPC methods from `shhext` API which are relevant to this spec, as well as motivation (rationale section) -->

## Secure Transport

In order to provide confidentiality, integrity, authentication and forward
secrecy of messages we implement a secure transport on top of Whisper. This is
used in 1:1 chats and group chats, but not for public chats. See [Status Secure
Transport Spec](status-secure-transport-spec.md) for more.

## Data Sync

[MVDS](https://specs.vac.dev/mvds.html) is used for 1:1 and group chats, however it is currently not in use for public chats.

<!-- TODO: Provide more detail on how this integrates with the other layers -->

## Payloads and clients

On top of secure transport, we have various types of data sync clients and
payload formats for things like 1:1 chat, group chat and public chat. These have
various degrees of standardization. Please refer to [Initial Message Payload
Specification](x8.md) for more details.

## Design Rationale

### P2P Overlay

#### Why devp2p? Why not use libp2p?

At the time the main Status clients were being developed, devp2p was the most
mature. However, it is likely we'll move over to libp2p in the future, as it'll
provide us with multiple transports, better protocol negotiation, NAT traversal,
etc.

For very experimental bridge support, see the bridge between libp2p and devp2p
in [Murmur](https://github.com/status-im/murmur).

#### What about other RLPx subprotocols like LES, and Swarm?

Status is primarily optimized for resource restricted devices, and at present
time light client support for these protocols are suboptimal. This is a work in
progress.

For better Ethereum light client support, see [Re-enable LES as
option](https://github.com/status-im/status-go/issues/1025). For better Swarm
support, see [Swarm adaptive
nodes](https://github.com/ethersphere/SWIPs/pull/12).

For transaction support, Status clients currently have to rely on Infura.

Status clients currently do not offer native support for file storage.

#### Why do you use Whisper?

Whisper is one of the [three parts](http://gavwood.com/dappsweb3.html) of the
vision of Ethereum as the world computer, Ethereum and Swarm being the other
two. Status was started as an encapsulation of and a clear window to this world
computer.

#### I heard you were moving away from Whisper?

Whisper is not currently under active development, and it has several drawbacks.
Among others:

- It is very wasteful bandwidth-wise and it doesn't appear to be scalable
- Proof of work is a poor spam protection mechanism for heterogenerous devices
- The privacy guarantees provided are not rigorous
- There's no incentives to run a node

Finding a more suitable transport privacy is an ongoing research effort,
together with [Vac](https://vac.dev/vac-overview) and other teams in the space.

#### Why is PoW for Whisper set so low?

A higher PoW would be desirable, but this kills the battery on mobilephones,
which is a prime target for Status clients.

This means the network is currently vulnerable to DDoS attacks. Alternative
methods of spam protection are currently being researched.

#### Why do you not use Discovery v5 for node discovery?

At the time we implemented dynamic node discovery, Discovery v5 wasn't completed
yet. Additionally, running a DHT on a mobile leads to slow node discovery, bad
battery and poor bandwidth usage. Instead, each client can choose to turn on
Discovery v5 for a short period of time until their peer list is populated.

For some further investigation, see
[here](https://github.com/status-im/swarms/blob/master/ideas/092-disc-v5-research.md).

## Footnotes

1. <https://github.com/status-im/status-protocol-go/>
1. <https://github.com/status-im/status-console-client/>
1. <https://github.com/status-im/status-react/>

## Acknowledgements
