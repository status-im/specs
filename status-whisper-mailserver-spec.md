# Status Whisper Mailserver Specification
> Version: 0.1 (Draft)
>
> Authors: Adam Babik <adam@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Abstract

Status clients are often offline. In order to allow clients to talk to each other while one is offline, we provide offline inboxing.

This current specification is an extension of Whisper v6 and operates under a store-and-forward model.

## Table of Contents

TBD.

## Introduction

In the case of mobile clients which are often offline, there is a strong need to have an ability to download offline messages. By offline messages, we mean messages sent into the Whisper network and expired before being collected by the recipient. A message stays in the Whisper network for a duration specified as `TTL` (time-to-live) property.

## Mailserver

A mailserver can either be running as a server or as a client.

### Server

<!-- TODO: This doesn't actually describe how to implement a mailserver -->

`MailServer` is an interface with two methods:
* `Archive(env *Envelope)`
* `DeliverMail(whisperPeer *Peer, request *Envelope)`

### Client

A Whisper client needs to register a mail server instance which will be used by [geth's Whisper service](https://github.com/ethereum/go-ethereum/blob/v1.8.23/whisper/whisperv6/whisper.go#L209-L213).

If a mail server is registered for a given Whisper client, it will save all incoming messages on a local disk (this is the simplest implementation, it can store the messages wherever it wants, also using technologies like swarm and IPFS) in the background.

Notice that each node is meant to be independent and SHOULD keep a copy of all historic messages. High Availability (HA) can be achieved by having multiple nodes in different locations. Additionally, each node is free to store messages in a way which provides storage HA as well.

Saved messages are delivered to a requester (another Whisper peer) asynchronously as a response to `p2pMessageCode` message code. This is not exposed as a JSON-RPC method in `shh` namespace but it's exposed in status-go as `shhext_requestMessages` and blocking `shh_requestMessagesSync`. Read more about [Whisper V6 extensions](#whisper-v6-extensions-or-status-whisper-node).

In order to receive historic messages from a filter, p2p messages MUST be allowed when creating the filter. Receiving p2p messages is implemented in [geth's Whisper V6 implementation](https://github.com/ethereum/go-ethereum/blob/v1.8.23/whisper/whisperv6/whisper.go#L739-L751).

## Security considerations

### Confidentiality

All Whisper envelopes are encrypted, and a mailserver node can't inspect their contents.

### High-availability

Since mailservers rely on being online to receive messages on behalf of other clients, this puts a high-availability requirement on individual nodes.

In practice, it is best to treat individual nodes as a form of a cache, and ensure consistency of messages at a different layer. See data sync layer.

### Altruistic and centralized operator risk

TBD.

### Privacy concerns

In order to use a mail server, a given node needs to connect to it directly, i.e. add the mail server as its peer and mark it as trusted. This means that the mail server is able to send direct p2p messages to the node instead of broadcasting them. Effectively, it knows which topics the node is interested in, when it is online as well as many metadata like IP address.

### Denial-of-service

TBD.
