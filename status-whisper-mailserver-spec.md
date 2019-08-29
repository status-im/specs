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

TBD.

## Mailserver

TBD.

### Server

TBD.

### Client

TBD.

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
