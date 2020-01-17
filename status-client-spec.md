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
- [Status Client Specification](#status-client-specification)
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
    - [BIPs and EIPs Standards support](#bips-and-eips-standards-support)
  - [Security Considerations](#security-considerations)
    - [Censorship-resistance](#censorship-resistance)
  - [Design Rationale](#design-rationale)
    - [P2P Overlay](#p2p-overlay-1)
      - [Why devp2p? Why not use libp2p?](#why-devp2p-why-not-use-libp2p)
      - [What about other RLPx subprotocols like LES, and Swarm?](#what-about-other-rlpx-subprotocols-like-les-and-swarm)
      - [Why do you use Whisper?](#why-do-you-use-whisper)
      - [I heard you were moving away from Whisper?](#i-heard-you-were-moving-away-from-whisper)
      - [Why is PoW for Whisper set so low?](#why-is-pow-for-whisper-set-so-low)
      - [Why do you not use Discovery v5 for node discovery?](#why-do-you-not-use-discovery-v5-for-node-discovery)
      - [I heard something about mailservers being trusted somehow?](#i-heard-something-about-mailservers-being-trusted-somehow)
    - [Data sync](#data-sync)
      - [Why is MVDS not used for public chats?](#why-is-mvds-not-used-for-public-chats)
  - [Footnotes](#footnotes)
  - [Appendix A: Security considerations](#appendix-a-security-considerations)
    - [Scalability and UX](#scalability-and-ux)
    - [Privacy](#privacy)
    - [Spam resistance](#spam-resistance)
    - [Censorship resistance](#censorship-resistance)
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

In order to retrieve the addresses of bootstrap nodes, a client MUST traverse
he merkle tree found at [`fleet.status.im`](https://fleet.status.im), as described in [EIP-1459](https://eips.ethereum.org/EIPS/eip-1459#client-protocol).

#### Discovery

To implement a Status client you need to discover peers to connect to. We use a
light discovery mechanism based on a combination of [Discovery
v5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md) and
[Rendezvous Protocol](https://github.com/libp2p/specs/tree/master/rendezvous),
(with some
[modifications](https://github.com/status-im/rendezvous#differences-with-original-rendezvous)).
Additionally, some static nodes MAY also be used.

A Status client MUST use at least one discovery method or use static nodes
to communicate with other clients.

Discovery V5 uses bootstrap nodes to discover other peers. Bootstrap nodes MUST support
Discovery V5 protocol as well in order to provide peers. It is kademlia-based discovery mechanism
and it might consume significant (at least on mobile) amount of network traffic to operate.

In order to take advantage from simpler and more mobile-friendly peers discovery mechanism,
i.e. Rendezvous protocol, one MUST provide a list of Rendezvous nodes which speak
Rendezvous protocol. Rendezvous protocol is request-response discovery mechanism.
It uses Ethereum Node Records (ENR) to report discovered peers.

Both peers discovery mechanisms use topics to provide peers with certain capabilities.
There is no point in returning peers that do not support a particular protocol.
Status nodes that want to be discovered MUST register to Discovery V5 and/or Rendezvous
with the `whisper` topic. Status nodes that are mail servers and want to
be discoverable MUST additionally register with the `whispermail` topic.

The recommended strategy is to use both mechanisms but at the same time implement a structure
called `PeerPool`. `PeerPool` is responsible for maintaining an optimal number of peers.
For mobile nodes, there is no significant advantage to have more than 2-3 peers and one mail server.
`PeerPool` can notify peers discovery protocol implementations that they should suspend
their execution because the optimal number of peers is found. They should resume
if the number of connected peers drops or a mail server disconnects.

It is worth noticing that an efficient caching strategy can be of great use, especially,
on mobile devices. Discovered peers can be cached as they rarely change and used
when the client starts again. In such a case, there might be no need to even start
peers discovery protocols because cached peers will satisfy the optimal number of peers.

Alternatively, a client MAY rely exclusively on a list of static peers. This is the most efficient
way because there is no peers discovery algorithm overhead introduced. The disadvantage
is that these peers might be gone and without peers discovery mechanism, it won't be possible to find
new ones.

The current list of static peers is published on https://fleets.status.im/. `eth.beta` is the current
group of peers the official Status client uses. The others are test networks.

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
Specification](status-payloads-spec.md) for more details.

### BIPs and EIPs Standards support

For a list of EIPs and BIPs that SHOULD be supported by Status client, please
see [Status EIPs Standards](status-EIPs.md).

## Security Considerations

TBD.

<!-- TODO: Fill this out. -->

### Censorship-resistance

With default settings Whisper over DevP2P runs on odd ports in 30k range, which
are easy to block. One workaround for this is to run ports on 443. This doesn't
take care of all cases though, and this quickly leads into efforts such as
obfuscated transports a la Tor.

See https://github.com/status-im/status-react/issues/6351 for some discussion.

<!-- TODO: More detail on interop of ports and what we do precisely -->

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

In order to use a mail server, a given node needs to connect to it directly, i.e. add the mailserver as its peer and mark it as trusted. This means that the mail server is able to send direct p2p messages to the node instead of broadcasting them. Effectively, it knows the bloom filter of the topics the node is interested in, when it is online as well as many metadata like IP address.

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
2. <https://github.com/status-im/status-console-client/>
3. <https://github.com/status-im/status-react/>

## Appendix A: Security considerations

There are several security considerations to take into account when running Status. Chief among them are: scalability, DDoS-resistance and privacy. These also vary depending on what capabilities are used, such as mailserver, light node, and so on.

### Scalability and UX

**Bandwidth usage:**

In version 1 of Status, bandwidth usage is likely to be an issue. For more investigation into this, see [the theoretical scaling model](https://github.com/vacp2p/research/tree/dcc71f4779be832d3b5ece9c4e11f1f7ec24aac2/whisper_scalability).

**Mailserver High Availability requirement:**

A mailserver has to be online to receive messages for other nodes, this puts a high availability requirement on it.

**Gossip-based routing:**

Use of gossip-based routing doesn't necessarily scale. It means each node can see a message multiple times, and having too many light nodes can cause propagation probability that is too low. See [Whisper vs PSS](https://our.status.im/whisper-pss-comparison/) for more and a possible Kademlia based alternative.

**Lack of incentives:**

Status currently lacks incentives to run nodes, which means node operators are more likely to create centralized choke points.

### Privacy

**Light node privacy:**

The main privacy concern with light nodes is that directly connected peers will know that a message originates from them (as it are the only ones it sends). This means nodes can make assumptions about what messages (topics) their peers are interested in.

**Bloom filter privacy:**

By having a bloom filter where only the topics you are interested in are set, you reveal which messages you are interested in. This is a fundamental tradeoff between bandwidth usage and privacy, though the tradeoff space is likely suboptimal in terms of the [Anonymity](https://eprint.iacr.org/2017/954.pdf) [trilemma](https://petsymposium.org/2019/files/hotpets/slides/coordination-helps-anonymity-slides.pdf).

**Mailserver client privacy:**

A mailserver client has to trust a mailserver, which means they can send direct traffic. This reveals what topics / bloom filter a node is interested in, along with its peerID (with IP).

**Privacy guarantees not rigorous:**

Privacy for Whisper hasn't been studied rigorously for various threat models like global passive adversary, local active attacker, etc. This is unlike e.g. Tor and mixnets.

**Topic hygiene:**

Similar to bloom filter privacy, if you use a very specific topic you reveal more information. See scalability model linked above.

### Spam resistance

**PoW bad for heterogenerous devices:**

Proof of work is a poor spam prevention mechanism. A mobile device can only have a very low PoW in order not to use too much CPU / burn up its phone battery. This means someone can spin up a powerful node and overwhelm the network.

**Mailserver trusted connection:**

A mailserver has a direct TCP connection, which means they are trusted to send traffic. This means a malicious or malfunctioning mailserver can overwhelm an individual node.

### Censorship resistance

**Devp2p TCP port blockable:**

By default Devp2p runs on port `30303`, which is not commonly used for any other service. This means it is easy to censor, e.g. airport WiFi. This can be mitigated somewhat by running on e.g. port `80` or `443`, but there are still outstanding issues. See libp2p and Tor's Pluggable Transport for how this can be improved.

## Acknowledgements
