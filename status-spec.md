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

## Node discovery and roles

- Bootstrap nodes
- Whisper relayers
- Mailservers
- Mobile nodes (Status Clients)

## Design Rationale

## Footnotes

1. <https://github.com/status-im/status-protocol-go/>
1. <https://github.com/status-im/status-console-client/>
1. <https://github.com/status-im/status-react/>

## Acknowledgements
