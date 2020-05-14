---
permalink: /spec/2
parent: Stable specs
title: 2/ACCOUNT
---

# 2/ACCOUNT

> Version: 0.2
> 
> Status: Stable
>
> Authors: Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im> (alphabetical order)

## Abstract

In this specification we explain what Status account is, and how trust is established.

## Table of Contents

- [Abstract](#abstract)
- [Table of Contents](#table-of-contents)
- [Introduction](#introduction)
- [Initial Key Generation](#initial-key-generation)
    - [Public/Private Keypairs](#publicprivate-keypairs)
    - [X3DH Prekey bundle creation](#x3dh-prekey-bundle-creation)
- [Account Broadcasting](#account-broadcasting)
    - [X3DH Prekey bundles](#x3dh-prekey-bundles)
- [Optional Account additions](#optional-account-additions)
    - [ENS Username](#ens-username)
    <!-- - [User Profile Picture](#user-profile-picture) -->
- [Trust establishment](#trust-establishment)
    - [Terms Glossary](#terms-glossary)
    - [Contact Discovery](#contact-discovery)
        - [Public channels](#public-channels)
        - [Private 1:1 messages](#private-11-messages)
    - [Initial Key Exchange](#initial-key-exchange)
        - [Contact Request](#contact-request)
        - [Bundles](#bundles)
    - [Contact Verification](#contact-verification)
        - [Identicon](#identicon)
        - [3 word pseudonym / Whisper/Waku key fingerprint](#3-word-pseudonym--whisperwaku-key-fingerprint)
        - [ENS name](#ens-name)
- [Security Considerations](#security-considerations)

<!-- markdown-toc end -->

## Introduction

The core concept of an account in Status is a set of cryptographic keypairs. Namely, the combination of the following:
1. a Whisper/Waku chat identity keypair
1. a set of cryptocurrency wallet keypairs

Everything else associated with the contact is either verified or derived from the above items, including:
- Ethereum address (future verification, currently the same base keypair)
- 3 word mnemonic name
- identicon
- message signatures

## Initial Key Generation
### Public/Private Keypairs 
- An ECDSA (secp256k1 curve) public/private keypair MUST be generated via a [BIP43](https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki) derived path from a [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) mnemonic seed phrase.
- The default paths are defined as such:
    - Whisper/Waku Chat Key (`IK`): `m/43'/60'/1581'/0'/0`  (post Multiaccount integration)
        - following [EIP1581](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1581.md)
    <!-- WE CURRENTLY DO NOT IMPLEMENT ENCRYPTION KEY, FOR FUTURE - C.P. -->
    <!-- - DB encryption Key (`DBK`): `m/43'/60'/1581'/1'/0` (post Multiaccount integration)
        - following [EIP1581](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1581.md) -->
    - Status Wallet paths: `m/44'/60'/0'/0/i` starting at `i=0`
        - following [BIP44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki)
        - NOTE: this (`i=0`) is also the current (and only) path for Whisper/Waku key before Multiaccount integration

### X3DH Prekey bundle creation
- Status follows the X3DH prekey bundle scheme that [Open Whisper Systems](https://en.wikipedia.org/wiki/Signal_Messenger#2013%E2%80%932018:_Open_Whisper_Systems) (not to be confused with the Whisper sub-protocol) outlines [in their documentation](https://signal.org/docs/specifications/x3dh/#the-x3dh-protocol) with the following exceptions:
    - Status does not publish one-time keys `OPK` or perform DH including them, because there are no central servers in the Status implementation. 
- A client MUST create X3DH prekey bundles, each defined by the following items:
    - Identity Key: `IK`
    - Signed prekey: `SPK`
    - Prekey signature: `Sig(IK, Encode(SPK))`
    - Timestamp
- These bundles are made available in a variety of ways, as defined in section 2.1.

## Account Broadcasting
- A user is responsible for broadcasting certain information publicly so that others may contact them.

### X3DH Prekey bundles
- A client SHOULD regenerate a new X3DH prekey bundle every 24 hours.  This MAY be done in a lazy way, such that a client that does not come online past this time period does not regenerate or broadcast bundles.
- The current bundle SHOULD be broadcast on a Whisper/Waku topic specific to his Identity Key, `{IK}-contact-code`, intermittently.  This MAY be done every 6 hours.
- A bundle SHOULD accompany every message sent.
- TODO: retreival of long-time offline users bundle via `{IK}-contact-code` 

## Optional Account additions

### ENS Username
- A user MAY register a public username on the Ethereum Name System (ENS).  This username is a user-chosen subdomain of the `stateofus.eth` ENS registration that maps to their Whisper/Waku identity key (`IK`). 

<!-- ### User Profile Picture
- An account MAY edit the `IK` generated identicon with a chosen picture.  This picture will become part of the publicly broadcasted profile of the account. -->

<!-- TODO: Elaborate on wallet account and multiaccount -->
<!-- TODO: Elaborate on security implications -->

## Trust establishment

**Trust establishment deals with users verifying they are communicating with who they think they are.**

### Terms Glossary

| term             | description |
| ---------------- | ----------- |
| privkey          | ECDSA secp256k1 private key |
| pubkey           | ECDSA secp256k1 public key |
| Whisper/Waku key | pubkey for chat with HD derivation path m/43'/60'/1581'/0'/0 |


### Contact Discovery

#### Public channels
- Public group channels in Status are a broadcast/subscription system.  All public messages are encrypted with a symmetric key drived from the channel name, `K_{pub,sym}`, which is publicly known.
- A public group channel's symmetric key MUST creation must follow the [web3 API](https://web3js.readthedocs.io/en/1.0/web3-shh.html#generatesymkeyfrompassword)'s `web3.ssh.generateSymKeyFromPassword` function
- In order to post to a public group channel, a client MUST have a valid account created.
- In order to listen to a public group channel, a client must subscribe to the channel name.  The sender of a message is derived from the message's signature.
- Discovery of channel names is not currently part of the protocol, and is typically done out of band.  If a channel name is used that has not been used, it will be created.
- A client MUST sign the message otherwise it will be discarded by the recipients.
- channel name specification:
    - matches `[a-z0-9\-]`
    - is not a public key

#### Private 1:1 messages
This can be done in the following ways:
1. scanning a user generated QR code
1. discovery through the Status app
1. asyncronous X3DH key exchange
1. public key via public channel listening
    - `status-react/src/status_im/contact_code/core.cljs`
1. contact codes
1. decentralized storage (not implemented)
1. Whisper/Waku

### Initial Key Exchange

#### Contact Request

#### Bundles
- An X3DH prekey bundle is defined as ([code](https://github.com/status-im/status-go/messaging/chat/protobuf/encryption.pb.go)):
  ```
  Identity                // Identity key
  SignedPreKeys           // a map of installation id to array of signed prekeys by that installation id
  Signature               // Prekey signature
  Timestamp               // When the bundle was lasted created locally
  ```
  - include BundleContainer???
- a new bundle SHOULD be created at least every 12 hours
- a bundle is only generated when it is used
- a bundle SHOULD be distributed on the contact code channel. This is the Whisper and Waku topic `{IK}-contact-code`, where `IK` is the hex encoded public key of the user, prefixed with `0x`. The channel is encrypted in the same way public chats are encrypted.

```js
// TODO is "include BundleContainer???" supposed to have "???"
```

### Contact Verification

Once you have the information of a contact, the following can be used to verify that the key material is as it should be.

#### Identicon
A low-poly identicon is deterministically generated from the Whisper/Waku chat public key.  This can then be compared out of band to ensure the reciever's public key is the one you have locally.

#### 3 word pseudonym / Whisper/Waku key fingerprint
Status generates a deterministic 3-word random pseudonym from the Whisper/Waku chat public key.  This pseudonym acts as a human readable fingerprint to the Whisper/Waku chat public key.  This name also shows when viewing a contact's public profile and in the chat UI.
- implementation: [gfycat](https://github.com/status-im/status-react/tree/develop/src/status_im/utils/gfycat)

#### ENS name
Status offers the ability to register a mapping of a human readable subdomain of `stateofus.eth` to their Whisper/Waku chat public key.  This registration is purchased (currently by staking 10 SNT) and stored on the Ethereum mainnet blockchain for public lookup.

<!-- TODO: Elaborate on security implications -->

<!-- TODO: Incorporate or cut below into proper spec


### Possible Connection Breakdown

possible connections
- client - client (not really ever, this is facilitated through all other connections)
    - personal chat
        - ratcheted with X3DH
    - private group chat
        - pairwise ratcheted with X3DH
    - public chat
- client - mailserver (statusd + ???)
    - a mailserver identifies itself by an [enode address](https://github.com/ethereum/wiki/wiki/enode-url-format) 
- client - Whisper/Waku node (statusd)
    - a node identifies itself by an enode address
- client - bootnode (geth)
    - a bootnode identifies itself by
        - an enode address
        - `NOTE: redezvous information here`
- client - ENS registry (ethereum blockchain -> default to infura)
- client - Ethereum RPC (custom geth RPC API -> default to infura API)
- client - IPFS (Status hosted IPFS gateway -> defaults to ???)
    - we have a status hosted IPFS gateway for pinning but it currently isn't used much.

### Notes

A user in the system is a public-private key pair using the Elliptic-Curve Cryptography secp256k1 that Ethereum uses.
- A 3-word random name is derived from the public key using the following package
    - `NOTE: need to find package`
    - This provides an associated human-readble fingerprint to the user's public key
- A user can optionally add additional layers on top of this keypair
    - Chosen username
    - ENS username

All messages sent are encrypted with the public key of the destination and signed by the private key of the given user using the following scheme:
- private chat
    - X3DH is used to define shared secrets which is then double ratcheted
- private group chat
    - considered pairwise private chats 
- public group chat
    - the message is encrypted with a symmetric key derived from the chat name

-->

## Security Considerations

-
