# Status Client Specification

> Version: 0.1 (Draft)
>
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

## Table of Contents

- [Abstract](#abstract)
- [P2P Overlay](#p2p-overlay)
- [Node discovery and roles](#node-discovery-and-roles)
- [Design Rationale](#design-rationale)
    - [P2P Overlay](#p2p-overlay-1)
        - [Why devp2p? Why not use libp2p?](#why-devp2p-why-not-use-libp2p)
        - [What about other RLPx subprotocols like LES, and Swarm?](#what-about-other-rlpx-subprotocols-like-les-and-swarm)
        - [Why do you use Whisper?](#why-do-you-use-whisper)
        - [I heard you were moving away from Whisper?](#i-heard-you-were-moving-away-from-whisper)
- [Footnotes](#footnotes)
- [Acknowledgements](#acknowledgements)

## Abstract

In this specification, we describe how to write a Status client for
communicating with other Status clients.

We present a reference implementation of the protocol <sup>1</sup> that is used
in a command line client <sup>2</sup> and a mobile app <sup>3</sup>.

This document consists of two parts. The first outlines the specifications that
have to be implemented in order to be a full Status client.

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

<!-- TODO: Add a link to Discovery v5 / Rendezvous spec -->

### Mailservers

Whisper Mailservers are special nodes that helps with offline inboxing. This
means you can receive Whisper envelopes after they have expired, which is useful
if they are sent while a node is offline. They operate on a store-and-forward
model.

<!-- TODO: Add a link to mailserver spec spec -->

### Mobile nodes

This is a Whisper node which connects to part of the Whisper network. It MAY
relay messages, and it MUST set a PoW requirement of at least `0.01` to be
accepted by other Status nodes.

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

## Footnotes

1. <https://github.com/status-im/status-protocol-go/>
1. <https://github.com/status-im/status-console-client/>
1. <https://github.com/status-im/status-react/>

## Acknowledgements
