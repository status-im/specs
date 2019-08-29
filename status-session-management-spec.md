# Status Session Management Specification

> Version: 0.1 (Draft)
>
> Authors: Dean Eigenmann <dean@status.im>, Andrea Piana <andreap@status.im>, Pedro Pombeiro <pedro@status.im>, Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im>

## Abstract

In this specification we describe how status sessions are handled.

<!-- TODO: Clarify what we mean by a session -->

## Table of Contents
  - [Abstract](#abstract)
  - [Introduction](#introduction)
  - [Initialization](#initialization)
  - [Concurrent sessions](#concurrent-sessions)
  - [Re-keying](#re-keying)
  - [Multi-device support](#multi-device-support)
  - [Pairing](#pairing)
  - [Sending messages to a paired group](#sending-messages-to-a-paired-group)
  - [Account recovery](#account-recovery)
  - [Partitioned devices](#partitioned-devices)
  - [Trust establishment](#trust-establishment)
    - [Contact request](#contact-request)
  - [Expired session](#expired-session)
  - [Stale devices](#stale-devices)

## Introduction

A peer is identified by two pieces of data:

1) An `installation-id` which is generated upon creating a new account in the `Status` application
2) Their identity whisper key

## Initialization

A new session is initialized once a successful X3DH exchange has taken place. Subsequent messages will use the established session until re-keying is necessary.

## Concurrent sessions

If two sessions are created concurrently between two peers the one with the symmetric key, first in byte order should be used this marks that the other has expired.

## Re-keying

On receiving a bundle from a given peer with a higher version, the old bundle should be marked as expired and a new session should be established on the next message sent.

## Multi-device support

Multi-device support is quite challenging as we don't have a central place where information on which and how many devices (identified by their respective `installation-id`) belongs to a whisper-identity.

Furthermore we always need to take account recovery in consideration, where the whole device is wiped clean and all the information about any previous sessions is lost.

Taking these considerations into account, the way multi-device information is propagated through the network is through bundles/contact codes, which will contain information about paired devices as well as information about the sending device.

This mean that every time a new device is paired, the bundle needs to be updated and propagated with the new information, and the burden is put on the user to make sure the pairing is successful.

The method is loosely based on https://signal.org/docs/specifications/sesame/ .

<!-- TODO: This multi device section isn't clear enough -->
<!-- TODO: Additionally, it seems tightly coupled with secure transport, which makes things like multi device public chats harder to reason about (IMO). E.g. as a client impl I might want multi device support but not want to impl double ratchet etc, so what does this mean? -->

## Pairing

When a user adds a new account in the `Status` application, a new `installation-id` will be generated. The device should be paired as soon as possible if other devices are present. Once paired the contacts will be notified of the new device and it will be included in further communications.

Any time a bundle from your `IK` but different `installation-id` is received, the device will be shown to the user and will have to be manually approved, to a maximum of 3. Once that is done any message sent by one device will also be sent to any other enabled device.

Once a new device is enabled, a new contact-code/bundle will be generated which will include pairing information.

The bundle will be propagated to contacts through the usual channels.

Removal of paired devices is a manual step that needs to be applied on each device, and consist simply in disabling the device, at which point pairing information will not be propagated anymore.

## Sending messages to a paired group

When sending a message, the peer will send a message to any `installation-id` that they have seen, using pairwise encryption, including their own devices.

The number of devices is capped to 3, ordered by last activity.

## Account recovery

Account recovery is no different from adding a new device, and it is handled in exactly the same way.

## Partitioned devices

In some cases (i.e. account recovery when no other pairing device is available, device not paired), it is possible that a device will receive a message that is not targeted to its own `installation-id`.
In this case an empty message containing bundle information is sent back, which will notify the receiving end of including this device in any further communication.

## Trust establishment

Trust establishment deals with users verifying they are communicating with who they think they are.

<!-- TODO: Deduplicate this and status accounts trust establishment -->

### Contact request

Once two accounts have been generated (Alice and Bob), Alice can send a contact request with an introductory message to Bob.

There are two possible scenarios, which dictate the presence or absence of a prekey bundle:
1. If Alice is using Bob's public chat key or ENS name, no prekey bundle is present;
1. If Alice found Bob through the app or scanned Bob's QR code, a prekey bundle is embedded and can be used to set up a secure channel as described in the [Initial key exchange flow X3DH](#initial-key-exchange-flow-X3DH) section.

Bob receives a contact request, informing him of:
- Alice's introductory message.

If Bob's prekey bundle was not available to Alice, Perfect Forward Secrecy hasn't yet been established. In any case, there are no implicit guarantees that Alice is whom she claims to be, and Bob should perform some form of external verification (e.g., using an Identicon).

If Bob accepts the contact request, a secure channel is created (if it wasn't already), and a visual indicator is displayed to signify that PFS has been established. Bob and Alice can then start exchanging messages, making use of the Double Ratchet algorithm as explained in more detail in [Double Ratchet](#double-ratchet) section.
If Bob denies the request, Alice is not able to send messages and the only action available is resending the contact request.

## Expired session

Expired session should not be used for new messages and should be deleted after 14 days from the expiration date, in order to be able to decrypt out-of-order and mailserver messages.

## Stale devices

When a bundle is received from `IK` a timer is initiated on any `installation-id` belonging to `IK` not included in the bundle. If after 7 days no bundles are received from these devices they are marked as `stale` and no message will be sent to them.
