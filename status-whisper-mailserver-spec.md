# Status Whisper Mailserver Specification
> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

- [Status Whisper Mailserver Specification](#status-whisper-mailserver-specification)
  - [Abstract](#abstract)
  - [Mailserver](#mailserver)
    - [Archiving messages](#archiving-messages)
    - [Delivering messages](#delivering-messages)
  - [Security considerations](#security-considerations)
    - [Confidentiality](#confidentiality)
    - [Altruistic and centralized operator risk](#altruistic-and-centralized-operator-risk)
    - [Privacy concerns](#privacy-concerns)
    - [Denial-of-service](#denial-of-service)

## Abstract

Being mostly offline is an intrinsic property of mobile clients. They need to save network transfer and battery consumption to avoid spending too much money or constant charging. Whisper protocol, on the other hand, is an online protocol. Messages are available in the Whisper network only for short period of time calculate in seconds.

Whisper Mailserver is a Whisper extension that allows to store messages permamently and deliver them to the clients even though they are already not available in the network and expired.

## Mailserver

From the network perspective, Mailserver is just like any other Whisper node. The only different is that it has a capability of archiving messages and delivering them to its peers on-demand.

It is important to notice that Mailserver will only handle requests from its direct peers and exchanged packets between Mailserver and a peer are p2p messages.

### Archiving messages

In order to store messages, one MUST implement the interface below and MUST register it within a Whisper service. The only known Whisper implementation that allows that is [geth](https://github.com/ethereum/go-ethereum).

`MailServer` interface consist of:
* `Archive(env *Envelope)`
* `DeliverMail(whisperPeer *Peer, request *Envelope)`

### Delivering messages

Mailserver delivers archieved messages to a peer after receiving a Whisper packet with code `p2pRequestCode`. Messages are delivered asynchronously, i.e. a requester sends a Whisper packet with code `p2pRequestCode` and at some point later, it will start receiving Whisper packets with code `p2pMessageCode`.

How a peer can initialize the request to a Mailserver is up to the implementator. Status peers acting as Mailserver expose two additional JSON-RPC methods: `shhext_requestMessages` and `shh_requestMessagesSync`.

Because all packets exchanged between a Mailserver and a peer are p2p packets, all filters created by a peer from which it expectes to receive archived messages MUST allow processing of direct peer-to-peer messages.

## Security considerations

### Confidentiality

All Whisper envelopes are encrypted. Mailserver node can not inspect their contents.

### Altruistic and centralized operator risk

In order to be useful, a mailserver SHOULD be online most of time. That means
you either have to be a bit tech-savvy to run your own node, or rely on someone
else to run it for you.

Currently Status Gmbh provides mailservers in an altruistic manner, but this is
suboptimal from a decentralization, continuance and risk point of view. Coming
up with a better system for this is ongoing research.

A Status client SHOULD allow the mailserver selection to be customizable.

### Privacy concerns

In order to use a Mailserver, a given node needs to connect to it directly,
i.e. add the Mailserver as its peer and mark it as trusted. This means that the
Mailserver is able to send direct p2p messages to the node instead of
broadcasting them. Effectively, it knows which topics the node is interested in,
when it is online as well as many metadata like IP address.

### Denial-of-service

Since a mailserver is delivering expired envelopes and has a direct TCP connection with the recipient, the recipient is vulnerable to DoS attacks from a malicious mailserver node.
