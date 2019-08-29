# Status Secure Transport Specification

> Version: 0.1 (Draft)
>
> Author: Andrea Piana <andreap@status.im>, Pedro Pombeiro <pedro@status.im>, Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im>

## Abstract

This document describes how we provide a secure channel between two peers, and thus provide confidentiality, integrity, authenticity and forward secrecy. It is transport-agnostic and works over asynchronous networks.

It builds on the [X3DH](https://signal.org/docs/specifications/x3dh/) and [Double Ratchet](https://signal.org/docs/specifications/doubleratchet/) specifications, with some adaptations to operate in a decentralized environment.

## Table of Contents

- [Abstract](#abstract)
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
    - [Definitions](#definitions)
    - [Design Requirements](#design-requirements)
    - [Conventions](#conventions)
    - [Transport Layer](#transport-layer)
    - [User flow for 1-to-1 communications](#user-flow-for-1-to-1-communications)
        - [Account generation](#account-generation)
        - [Account recovery](#account-recovery)
- [Messaging](#messaging)
    - [End-to-end encryption](#end-to-end-encryption)
    - [Prekeys](#prekeys)
    - [Bundle retrieval](#bundle-retrieval)
    - [1:1 chat contact request](#11-chat-contact-request)
        - [Initial key exchange flow (X3DH)](#initial-key-exchange-flow-x3dh)
        - [Double Ratchet](#double-ratchet)
- [Session Management](#session-management)
- [Initialization](#initialization)
- [Concurrent sessions](#concurrent-sessions)
- [Re-keying](#re-keying)
- [Multi-device support](#4-multi-device-support)
    - [Pairing](#pairing)
    - [Sending messages to a paired group](#sending-messages-to-a-paired-group)
    - [Account recovery](#account-recovery)
    - [Partitioned devices](#partitioned-devices)
- [Trust establishment](#trust-establishment)
- [Expired session](#expired-session)
- [Stale devices](#stale-devices)
- [Security Considerations](#security-considerations)

## Introduction

In this document we describe how a secure channel is established, and how various conversational security properties are achieved.

### Definitions

- **Perfect Forward Secrecy** is a feature of specific key-agreement protocols which provide assurances that your session keys will not be compromised even if the private keys of the participants are compromised. Specifically, past messages cannot be decrypted by a third-party who manages to get a hold of a private key.

- **Secret channel** describes a communication channel where Double Ratchet algorithm is in use.

### Design Requirements

- **Confidentiality**: The adversary should not be able to learn what data is being exchanged between two Status clients.
- **Authenticity**: The adversary should not be able to cause either endpoint of a Status 1:1 chat to accept data from any third party as though it came from the other endpoint.
- **Forward Secrecy**: The adversary should not be able to learn what data was exchanged between two Status clients if, at some later time, the adversary compromises one or both of the endpoint devices.

<!-- TODO: Integrity should be here -->
<!-- TODO: It is not clearly stated in this spec how we achieve confidentiality, authenticity and integrity. State this clearly. -->

### Conventions

Types used in this specification are defined using [Protobuf](https://developers.google.com/protocol-buffers/).

### Transport Layer

[Whisper](./status-whisper-usage-spec.md) serves as the transport layer for the Status chat protocol.

### User flow for 1-to-1 communications

#### Account generation

See [Account specification](./status-account-spec.md)

#### Account recovery

If Alice later recovers her account, the Double Ratchet state information will not be available, so she is no longer able to decrypt any messages received from existing contacts.

If an incoming message (on the same Whisper topic) fails to decrypt, a message is replied with the current bundle, so that the other end is notified of the new device. Subsequent communications will use this new bundle.

## Messaging

All 1:1 and group chat messaging in Status is subject to end-to-end encryption to provide users with a strong degree of privacy and security. Public chat messages are publicly readable by anyone since there's no permission model for who is participating in a public chat.

The rest of this document is purely about 1:1 and private group chat. Private group chat largely reduces to 1:1 chat, since there's a secure channel between each pair-wise participant.

### End-to-end encryption

End-to-end encryption (E2EE) takes place between two clients. The main cryptographic protocol is a [Status implementation](https://github.com/status-im/doubleratchet/) of the Double Ratchet protocol, which is in turn derived from the [Off-the-Record protocol](https://otr.cypherpunks.ca/Protocol-v3-4.1.1.html), using a different ratchet. The message payload is subsequently encrypted by the transport protocol - Whisper (see section [Transport Layer](#transport-layer)) -, using symmetric key encryption. 
Furthermore, Status uses the concept of prekeys (through the use of [X3DH](https://signal.org/docs/specifications/x3dh/)) to allow the protocol to operate in an asynchronous environment. It is not necessary for two parties to be online at the same time to initiate an encrypted conversation.

Status uses the following cryptographic primitives:
- Whisper
    - AES-256-GCM
    - KECCAK-256
- X3DH
    - Elliptic curve Diffie-Hellman key exchange (secp256k1)
    - KECCAK-256
    - ECDSA
    - ECIES
- Double Ratchet
    - HMAC-SHA-256 as MAC
    - Elliptic curve Diffie-Hellman key exchange (Curve25519)
    - AES-256-CTR with HMAC-SHA-256 and IV derived alongside an encryption key

    Key derivation is done using HKDF.

### Prekeys

Every client initially generates some key material which is stored locally:
- Identity keypair based on secp256k1 - `IK`
- A signed prekey based on secp256k1 - `SPK`
- A prekey signature - `Sig(IK, Encode(SPK))`

More details can be found in the `X3DH Prekey bundle creation` section of [Account specification](./status-account-spec.md#x3dh-prekey-bundle-creation).

A `contact-code` is a protobuf `Bundle` message, encoded in `JSON` and converted to their `base64` string representation.

Prekey bundles are can be extracted from any user's messages, or found via searching for their specific contact code topic, `{IK}-contact-code`.

TODO: See below on bundle retrieval, this seems like enhancement and parameter for recommendation

### Bundle retrieval
<!-- TODO: Potentially move this completely over to [Trust Establishment](./status-account-spec.md) -->

X3DH works by having client apps create and make available a bundle of prekeys (the X3DH bundle) that can later be requested by other interlocutors when they wish to start a conversation with a given user.

In the X3DH specification, a shared server is typically used to store bundles and allow other users to download them upon request. Given Status' goal of decentralization, Status chat clients cannot rely on the same type of infrastructure and must achieve the same result using other means. By growing order of convenience and security, the considered approaches are:
- contact codes;
- public and one-to-one chats;
- QR codes;
- ENS record;
- Decentralized permanent storage (e.g. Swarm, IPFS).
- Whisper

<!-- TODO: Comment, it isn't clear what we actually _do_. It seems as if this is exploring the problem space. From a protocol point of view, it might make sense to describe the interface, and then have a recommendation section later on that specifies what we do. See e.g. Signal's specs where they specify specifics later on.  -->

Since bundles stored in QR codes or ENS records cannot be updated to delete already used keys, the approach taken is to rotate more frequently the bundle (once every 24 hours), which will be propagated by the app through the channel available.

### 1:1 chat contact request

There are two phases in the initial negotiation of a 1:1 chat:
1. **Identity verification** (e.g., face-to-face contact exchange through QR code, Identicon matching). A QR code serves two purposes simultaneously - identity verification and initial bundle retrieval;
1. **Asynchronous initial key exchange**, using X3DH.

For more information on account generation and trust establishment, see [Status Account Specification](status-account-spec.md)

#### Initial key exchange flow (X3DH)

The initial key exchange flow is described in [section 3 of the X3DH protocol](https://signal.org/docs/specifications/x3dh/#sending-the-initial-message), with some additional context:
- The users' identity keys `IK_A` and `IK_B` correspond to their respective Status chat public keys;
- Since it is not possible to guarantee that a prekey will be used only once in a decentralized world, the one-time prekey `OPK_B` is not used in this scenario;
- Bundles are not sent to a centralized server, but instead served in a decentralized way as described in [bundle retrieval](#bundle-retrieval).

Bob's prekey bundle is retrieved by Alice, however it is not specific to Alice. It contains:

([protobuf](https://github.com/status-im/status-go/blob/a904d9325e76f18f54d59efc099b63293d3dcad3/services/shhext/chat/encryption.proto#L12))

``` protobuf
// X3DH prekey bundle
message Bundle {

  bytes identity = 1;

  map<string,SignedPreKey> signed_pre_keys = 2;

  bytes signature = 4;

  int64 timestamp = 5;
}
```
- `identity`: Identity key `IK_B`
- `signed_pre_keys`: Signed prekey `SPK_B` for each device, indexed by `installation-id`
- `signature`: Prekey signature <i>Sig(`IK_B`, Encode(`SPK_B`))</i>
- `timestamp`: When the bundle was created locally

([protobuf](https://github.com/status-im/status-go/blob/a904d9325e76f18f54d59efc099b63293d3dcad3/services/shhext/chat/encryption.proto#L5))

``` protobuf
message SignedPreKey {
  bytes signed_pre_key = 1;
  uint32 version = 2;
}
```

The `signature` is generated by sorting `installation-id` in lexicographical order, and concatenating the `signed-pre-key` and `version`:

`installation-id-1signed-pre-key1version1installation-id2signed-pre-key2-version-2`

#### Double Ratchet

Having established the initial shared secret `SK` through X3DH, we can use it to seed a Double Ratchet exchange between Alice and Bob.

Please refer to the [Double Ratchet spec](https://signal.org/docs/specifications/doubleratchet/) for more details.

The initial message sent by Alice to Bob is sent as a top-level `ProtocolMessage` ([protobuf](https://github.com/status-im/status-go/blob/a904d9325e76f18f54d59efc099b63293d3dcad3/services/shhext/chat/encryption.proto#L65)) containing a map of `DirectMessageProtocol` indexed by `installation-id` ([protobuf](https://github.com/status-im/status-go/blob/1ac9dd974415c3f6dee95145b6644aeadf02f02c/services/shhext/chat/encryption.proto#L56)):

``` protobuf
message ProtocolMessage {

  Bundle bundle = 1;

  string installation_id = 2;

  repeated Bundle bundles = 3;

  // One to one message, encrypted, indexed by installation_id
  map<string,DirectMessageProtocol> direct_message = 101;

  // Public chats, not encrypted
  bytes public_message = 102;

}
```

- `bundle`: optional bundle is exchanged with each message, deprecated;
- `bundles`: a sequence of bundles
- `installation_id`: the installation id of the sender
- `direct_message` is a map of `DirectMessageProtocol` indexed by `installation-id`
- `public_message`: unencrypted public chat message.

``` protobuf
message DirectMessageProtocol {
  X3DHHeader X3DH_header = 1;
  DRHeader DR_header = 2;
  DHHeader DH_header = 101;
  // Encrypted payload
  bytes payload = 3;
}
```
- `X3DH_header`: the `X3DHHeader` field in `DirectMessageProtocol` contains:

    ([protobuf](https://github.com/status-im/status-go/blob/a904d9325e76f18f54d59efc099b63293d3dcad3/services/shhext/chat/encryption.proto#L47))
    ``` protobuf
    message X3DHHeader {
      bytes key = 1;
      bytes id = 4;
    }
    ```

    - `key`: Alice's ephemeral key `EK_A`;
    - `id`: Identifier stating which of Bob's prekeys Alice used, in this case Bob's bundle signed prekey.

    Alice's identity key `IK_A` is sent at the transport layer level (Whisper);

- `DR_header`: Double ratchet header ([protobuf](https://github.com/status-im/status-go/blob/a904d9325e76f18f54d59efc099b63293d3dcad3/services/shhext/chat/encryption.proto#L31)). Used when Bob's public bundle is available:
    ``` protobuf
    message DRHeader {
      bytes key = 1;
      uint32 n = 2;
      uint32 pn = 3;
      bytes id = 4;
    }
    ```
    - `key`: Alice's current ratchet public key (as mentioned in [DR spec section 2.2](https://signal.org/docs/specifications/doubleratchet/#symmetric-key-ratchet));
    - `n`: number of the message in the sending chain;
    - `pn`: length of the previous sending chain;
    - `id`: Bob's bundle ID.

- `DH_header`: Diffie-Helman header (used when Bob's bundle is not available):
    ([protobuf](https://github.com/status-im/status-go/blob/a904d9325e76f18f54d59efc099b63293d3dcad3/services/shhext/chat/encryption.proto#L42))
    ``` protobuf
    message DHHeader {
      bytes key = 1;
    }
    ```
    - `key`: Alice's compressed ephemeral public key.

- `payload`:
    - if a bundle is available, contains payload encrypted with the Double Ratchet algorithm;
    - otherwise, payload encrypted with output key of DH exchange (no Perfect Forward Secrecy).

<!-- TODO: A lot of links to status-go, seems likely these should be updated to status-protocol-go -->

# Session Management

This section describe how sessions are handled.

A peer is identified by two pieces of data:

1) An `installation-id` which is generated upon creating a new account in the `Status` application
2) Their identity whisper key

## Initialization

A new session is initialized once a successful X3DH exchange has taken place.
Subsequent messages will use the established session until re-keying is necessary.

## Concurrent sessions

If two sessions are created concurrently between two peers the one with the symmetric key first in byte order should be used, marking the other has expired.

## Re-keying

On receiving a bundle from a given peer with a higher version, the old bundle should be marked as expired and a new session should be established on the next message sent.

## Multi-device support

Multi-device support is quite challenging as we don't have a central place where information on which and how many devices (identified by their respective `installation-id`) belongs to a whisper-identity.

Furthermore we always need to take account recovery in consideration, where the whole device is wiped clean and all the information about any previous sessions is lost.

Taking these considerations into account, the way multi-device information is propagated through the network is through bundles/contact codes, which will contain information about paired devices as well as information about the sending device.

This mean that every time a new device is paired, the bundle needs to be updated and propagated with the new information, and the burden is put on the user to make sure the pairing is successful.

The method is loosely based on https://signal.org/docs/specifications/sesame/ .

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

#### Contact request

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

# Security Considerations

The same considerations apply as in [section 4 of the X3DH spec](https://signal.org/docs/specifications/x3dh/#security-considerations) and [section 6 of the Double Ratchet spec](https://signal.org/docs/specifications/doubleratchet/#security-considerations), with some additions detailed below.

<!-- TODO: Add any additional context here not covered in the X3DH and DR specs -->

<!--
TODO: description here

### --- Security  and Privacy Features
#### Confidentiality (YES)
> Only the intended recipients are able to read a message. Specifically, the message must not be readable by a server operator that is not a conversation participant

- Yes.
- There's a layer of encryption at Whisper as well as above with Double Ratchet
- Relay nodes and Mailservers can only read a topic of a Whisper message, and nothing within the payload.

#### Integrity (YES)
> No honest party will accept a message that has been modified in transit.

- Yes.
- Assuming a user validates (TODO: Check this assumption) every message they are able to decrypt and validates its signature from the sender, then it is not able to be altered in transit.
    * [igorm] i'm really not sure about it, Whisper provides a signature, but I'm not sure we check it anywhere (simple grepping didn't give anything)
    * [andrea] Whisper checks the signature and a public key is derived from it, we check the public key is a meaningful public key. The pk itself is not in the content of the message for public chats/1-to-1 so potentially you could send a message from a random account without having access to the private key, but that would not be much of a deal, as you might just as easily create a random account)

#### Authentication (YES)
>  Each participant in the conversation receives proof of possession of a known long-term secret from all other participants that they believe to be participating in the conversation. In addition, each participant is able to verify that a message was sent from the claimed source

- 1:1 --- one-to-one messages are encrypted with the recipient's public key, and digitally signed by the sender's.  In order to provide Perfect Forward Secrecy, we build on the X3DH and Double Ratchet specifications from Open Whisper Systems, with some adaptations to operate in a decentralized environment.
- group --- group chat is pairwise
- public --- A user subscribes to a public channel topic and the decryption key is derived from the topic name

**TODO:** Need to verify that this is actually the case
**TODO:** Fill in explicit details here

#### Participant Consistency (YES?)
> At any point when a message is accepted by an honest party, all honest parties are guaranteed to have the same view of the participant list

- **TODO:** Need details here

#### Destination Validation (YES?)
> When a message is accepted by an honest party, they can verify that they were included in the set of intended recipients for the message.

- Users are aware of the topic that a message was sent to, and that they have the ability to decrypt it.
- 

#### Forward Secrecy (PARTIAL)
> Compromising all key material does not enable decryption of previously encrypted data

- After first back and forth between two contacts with PFS enabled, yes.

#### Backward Secrecy (YES)
> Compromising all key material does not enable decryption of succeeding encrypted data

- PFS requires both backward and forwards secrecy
[Andrea: This is not true, (Perfect) Forward Secrecy does not imply backward secrecy (which is also called post-compromise security, as signal calls it, or future secrecy, it's not well defined). Technically this is a NO , double ratchet offers good Backward secrecy, but not perfect. Effectively if all the key material is compromised, any future message received will be also compromised (due to the hash ratchet), until a DH ratchet step is completed (i.e. the compromised party generate a new random key and ratchet)]

#### Anonymity Preserving (PARTIAL)
> Any anonymity features provided by the underlying transport privacy architecture are not undermined (e.g., if the transport privacy system provides anonymity, the conversation security level does not de-anonymize users by linking key identifiers).

- by default, yes
- ENS Naming system attaches an identifier to a given public key

#### Speaker Consistency (PARTIAL)
> All participants agree on the sequence of messages sent by each participant. A protocol might perform consistency checks on blocks of messages during the protocol, or after every message is sent.

- We use Lamport timestamps for ordering of events.
- In addition to this, we use local timestamps to attempt a more intuitive ordering. [Andrea: currently this was introduced as a regression during performance optimization and might result in out-of-order messages if sent across day boundaries, so I consider it a bug and not part of the specs (it does not make the order more intuitive, quite the opposite as it might result in causally related messages being out-of-order, but helps dividing the messages in days)]
- Fundamentally, there's no single source of truth, nor consensus process for global ordering [Andrea: Global ordering does not need a consensus process i.e. if you order messages alphabetically, and you break ties consistently, you have global ordering, as all the participants will see the same ordering (as opposed to say order by the time the message was received locally),  of course is not useful, you want to have causal + global to be meaningful]

TODO: Understand how this is different from Global Transcript
[Andrea: This is basically Global transcript for a single participants, we offer global transcript]

#### Causality Preserving (PARTIAL)
> Implementations can avoid displaying a message before messages that causally precede it

- Not yet, but in pipeline (data sync layer)

[Andrea: Messages are already causally ordered, we don't display messages that are causally related out-of-order, that's already granted by lamport timestamps]

TODO: Verify if this can be done already by looking at Lamport clock difference

#### Global Transcript (PARTIAL)
> All participants see all messages in the same order

- See directly above

[Andrea: messages are globally (total) ordered, so all participants see the same ordering]

#### Message Unlinkability (NO)
> If a judge is convinced that a participant authored one message in the conversation, this does not provide evidence that they authored other messages

- Currently, the Status software signs every messages sent with the user's public key, thus making it no able to give unlinkability.  
- This is not necessary though, and could be built in to have an option to not sign. 
- Side note: moot account allows for this but is a function of the anonymity set that uses it.  The more people that use this account the stronger the unlinkability.

#### Message Repudiation (NO)
> Given a conversation transcript and all cryptographic keys, there is no evidence that a given message was authored by any particular user

- All messages are digitally signed by their sender.
- The underlying transport, Whisper, does allow for unsigned messages, but we don't use it.

#### Participant Repudiation (NO)
> Given a conversation transcript and all cryptographic key material for all but one accused (honest) participant, there is no evidence that the honest participant was in a conversation with any of the other participants.

### --- Group related features
#### Computational Equality (YES)
>  All chat participants share an equal computational load

- One a message is sent, all participants in a group chat perform the same steps to retrieve and decrypt it. 
- If proof of work is actually used at the Whisper layer (basically turned off in Status) then the sender would have to do additional computational work to send messages.

#### Trust Equality (PARTIAL)
> No participant is more trusted or takes on more responsibility than any other

- 1:1 chats and public chats are equal
- group chats have admins (on purpose)
- Private Group chats have Administrators and Members.  Upon construction, the creator is made an admin. These groups have the following privileges:
    - Admins:
        - Add group members
        - Promote group members to admin
        - Change group name
    - Members:
        - Accept invitation to group
        - Leave group
    - Non-Members:
        - Invited by admins show up as "invited" in group; this leaks contacat information
        - Invited people don't opt-in to being invited

TODO: Group chat dynamics should have a documented state diagram
TODO: create issues for identity leak of invited members as well as current members of a group showing up who have not accepted yet [Andrea: that's an interesting point, didn't think of that. Currently we have this behavior for 2 reasons, backward compatibility with previous releases, which had no concept of joining, and also because we rely on other peers to propagate group info, so we don't have a single-message point of failure (the invitation), the first can be addressed easily, the second is trickier, without giving up the propagation mechanism (if we choose to give this up, then it's trivial)]

#### Subgroup Messaging (NO)
> Messages can be sent to a subset of participants without forming a new conversation

- This would require a new topic and either a new public chat or a new group chat
[Andrea: This is a YES, as messages are pairwise encrypted, and client-side fanout, so anyone could potentially send a message only to a subset of the group]

#### Contractible Membership (PARTIAL)
> After the conversation begins, participants can leave without restarting the protocol

- For 1:1, there is no way to ignore or block a user from sending you a message.  This is currently in the pipeline.
- For public chats, Yes. A member simply stops subscribing to a specific topic and will no longer receive messages. 
- For group chats: this assumes pairwise encryption OR key is renegotiated
- This only currently works on the identity level, and not the device level.  A ghost device will have access to anything other devices have.
[Andrea: For group chats, that's possible as using pairwise encryption, also with group chats (which use device-to-device encryption), ghost devices is a bit more complicated, in general, they don't have access to the messages you send, i.e. If I send a message from device A1 to the group chat and there is a ghost device A2, it will not be able to decrypt the content, but will see that a message has been sent (as only paired devices are kept in sync, and those are explicitly approved by the user). Messages that you receive are different, so a ghost device (A2) will potentially be able to decrypt the message, but A1 can detect the ghost device (in most cases, it's complicated :), the pfs docs describe multi-device support), for one-to-one ghost devices are undetectable]

#### Expandable Membership (PARTIAL)
> After the conversation begins, participants can join without restarting the protocol.

- 1:1: no, only 1:1
- private group: yes, since it is pair-wise, each person in the group just creates a pair with the new member
- public: yes, as members of a public chat are only subscribing to a topic and receiving anyone sending messages to it. 

### --- Usability and Adoption

#### Out-of-Order Resilient (PARTIAL)
> If a message is delayed in transit, but eventually arrives, its contents are accessible upon arrival

- Due to asynchronous forward secrecy and no additional services, private keys might be rotated

[Andrea: That's correct, in some cases if the message is delayed for too long, or really out-of-order, the specific message key might have been deleted, as we only keep the last 3000 message keys]
[Igor: TTL of a whisper message can expire, so any node-in-transit will drop it. Also, I believe we ignore messages with skewed timestamps]

#### Dropped Message Resilient (PARTIAL)
> Messages can be decrypted without receipt of all previous messages. This is desirable for asynchronous and unreliable network services

- Public chats: yes, users are able to decrypt any message received at any time.
- 1-to-1/group chat also, this is a YES in my opinion

#### Asynchronous (PARTIAL)
> Messages can be sent securely to disconnected recipients and received upon their next connection

- The semantics around message reliability are currently poor
    * [Igor: messages are stored on mailservers for way longer than TTL (30 days), but that requires Status infrastructure]
- There's a TTL in Whisper and mailserver can deliver messages after the fact

TODO: this requires more detail

#### Multi-Device Support (YES)
> A user can participate in the conversation using multiple devices at once. Each device must be able to send and receive messages. Ideally, all devices have identical views of the conversation. The devices might use a synchronized long-term key or distinct keys.

- Yes
- There is currently work being done to improve the syncing process between a user's devices. 

#### No Additional Service (NO)
> The protocol does not require any infrastructure other than the protocol participants. Specifically, the protocol must not require additional servers for relaying messages or storing any kind of key material.

- The protocol requires whisper relay servers and mailservers currently. 
- The larger the number of whisper relay servers, the better the transport security but there might be potential scaling problems.
- Mailservers act to provide asynchronicity so users can retrieve messages after coming back from an offline period. 

-->
