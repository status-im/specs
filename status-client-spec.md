# Status Client Specification

> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Dean Eigenmann <dean@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Abstract

In this specification, we describe how to write a Status client for
communicating with other Status clients.

We present a reference implementation of the protocol <sup>1</sup> that is used
in a command line client <sup>2</sup> and a mobile app <sup>3</sup>.

This document consists of two parts. The first outlines the specifications that
have to be implemented in order to be a full Status client. The second gives a design rationale and answers some common questions.

## Table of Contents
- [Abstract](#abstract)
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
    - [Protocol layers](#protocol-layers)
- [Components](#components)
    - [P2P Overlay](#p2p-overlay)
        - [Node discovery and roles](#node-discovery-and-roles)
        - [Bootstrapping](#bootstrapping)
        - [Discovery](#discovery)
        - [Mobile nodes](#mobile-nodes)
    - [Transport privacy and Whisper usage](#transport-privacy-and-whisper-usage)
    - [Secure Transport](#secure-transport)
    - [Data Sync](#data-sync)
    - [Payloads and clients](#payloads-and-clients)
- [Security Considerations](#security-considerations)
- [Design Rationale](#design-rationale)
    - [P2P Overlay](#p2p-overlay-1)
        - [Why devp2p? Why not use libp2p?](#why-devp2p-why-not-use-libp2p)
        - [What about other RLPx subprotocols like LES, and Swarm?](#what-about-other-rlpx-subprotocols-like-les-and-swarm)
        - [Why do you use Whisper?](#why-do-you-use-whisper)
        - [I heard you were moving away from Whisper?](#i-heard-you-were-moving-away-from-whisper)
        - [Why is PoW for Whisper set so low?](#why-is-pow-for-whisper-set-so-low)
        - [Why do you not use Discovery v5 for node discovery?](#why-do-you-not-use-discovery-v5-for-node-discovery)
        - [I heard something about mailservers being trusted somehow?](#i-heard-something-about-mailservers-being-trusted-somehow)
- [Footnotes](#footnotes)
- [Acknowledgements](#acknowledgements)

## Introduction

### Protocol layers

Implementing a Status clients means implementing the following layers. Additionally, there are separate specifications for things like key management and account lifecycle.

| Layer             | Purpose                         | Technology                   |
|-------------------|---------------------------------|------------------------------|
| Data and payloads | End user functionality          | 1:1, group chat, public chat |
| Data sync         | Data consistency                | MVDS Ratchet                 |
| Secure transport  | Confidentiality, PFS, etc       | Double Ratchet               |
| Transport privacy | Routing, Metadata protection    | Whisper                      |
| P2P Overlay       | Overlay routing, NAT traversal  | devp2p                       |

## Components

### P2P Overlay

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

There MUST be an Ethereum node that is capable of discovering peers and
implements Whisper V6 specification.

#### Node discovery and roles

There are four types of node roles:
1. Bootstrap nodes
2. Whisper relayers
3. Mailservers (servers and clients)
4. Mobile nodes (Status Clients)

To implement a standard Status client you MUST implement the last node type. The
other node types are optional, but we RECOMMEND you implement a mailserver
client mode, otherwise the user experience is likely to be poor.

#### Bootstrapping

To connect to other Status nodes you need to connect to a bootstrap node. These
nodes allow you to discover other nodes of the network.

Currently the main bootstrap nodes are provided by Status Gmbh, but anyone can
run these provided they are connected to the rest of the Whisper network.

<!-- TODO: Add a link to bootstrap nodes -->

#### Discovery

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

#### Mobile nodes

This is a Whisper node which connects to part of the Whisper network. It MAY
relay messages. See next section for more details on how to use Whisper to
communicate with other Status nodes.

### Transport privacy and Whisper usage

Once a Whisper node is up and running there are some specific settings required
to commmunicate with other Status nodes.

See [Status Whisper Usage Spec](status-whisper-usage-spec.md) for more details.

For providing offline inboxing, see the complementary [Whisper Mailserver
Spec](status-whisper-mailserver-spec.md).

### Secure Transport

In order to provide confidentiality, integrity, authentication and forward
secrecy of messages we implement a secure transport on top of Whisper. This is
used in 1:1 chats and group chats, but not for public chats. See [Status Secure
Transport Spec](status-secure-transport-spec.md) for more.

### Data Sync

[MVDS](https://specs.vac.dev/mvds.html) is used for 1:1 and group chats, however it is currently not in use for public chats.

[Status payloads](#payloads-and-clients) are serialized and then wrapped inside a MVDS message which is added to an [MVDS payload](https://specs.vac.dev/mvds.html#payloads), this payload is then encrypted (if necessary for 1-to-1 / group-chats) and sent using whisper which also encrypts it.

### Payloads and clients

On top of secure transport, we have various types of data sync clients and
payload formats for things like 1:1 chat, group chat and public chat. These have
various degrees of standardization. Please refer to [Initial Message Payload
Specification](x8.md) for more details.

## Security Considerations

TBD.

<!-- TODO: Fill this out. -->

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

#### I heard something about mailservers being trusted somehow?

In order to use a mail server, a given node needs to connect to it directly, i.e. add the mailserver as its peer and mark it as trusted. This means that the mail server is able to send direct p2p messages to the node instead of broadcasting them. Effectively, it knows which topics the node is interested in, when it is online as well as many metadata like IP address.

### Data sync

#### Why is MVDS not used for public chats?

Currently public chats are broadcast-based, and there's no direct way of finding
out who is receiving messages. Hence there's no clear group sync state context
whereby participants can sync. Additionally, MVDS is currently not optimized for
large group contexts, which means bandwidth usage will be a lot higher than
reasonable. See [P2P Data Sync for
Mobile](https://vac.dev/p2p-data-sync-for-mobile) for more. This is an active
area of research. The bandwidth issue is further exacerbated by Whisper being
very bandwidth heavy.

## Footnotes

1. <https://github.com/status-im/status-protocol-go/>
1. <https://github.com/status-im/status-console-client/>
1. <https://github.com/status-im/status-react/>

## Acknowledgements
