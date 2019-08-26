# Status Client Specification

> Version: 0.1 (Draft)
>
> Authors: Oskar Thorén <oskar@status.im>, Dean Eigenmann <dean@status.im>

## Table of Contents

- [Abstract](#abstract)
- [Design Rationale](#design-rationale)
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
(here)[https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md]. See
more on node discovery and management in the next section.

To communicate between Ethereum nodes, the (RLPx Transport
Protocol, v5)[https://github.com/ethereum/devp2p/blob/master/rlpx.md] is used, which
allows for TCP-based communication between nodes.

On top of this we run the RLPx-based subprotocol (Whisper
v6)[https://eips.ethereum.org/EIPS/eip-627] for privacy-preserving messaging.

## Node discovery and roles

- Bootstrap nodes
- Whisper relayers
- Mailservers
- Mobile nodes (Status Clients)

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

For better Ethereum light client support, see (Re-enable LES as
option)[https://github.com/status-im/status-go/issues/1025]. For better Swarm
support, see (Swarm adaptive
nodes)[https://github.com/ethersphere/SWIPs/pull/12].

For transaction support, Status clients currently have to rely on Infura.

Status clients currently do not offer native support for file storage.

## Footnotes

1. <https://github.com/status-im/status-protocol-go/>
1. <https://github.com/status-im/status-console-client/>
1. <https://github.com/status-im/status-react/>

## Acknowledgements
