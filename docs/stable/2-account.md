---
permalink: /spec/2
parent: Stable specs
title: 2/ACCOUNT
---

# 2/ACCOUNT

> Version: 0.4
> 
> Status: Stable
>
> Authors: Corey Petty <corey@status.im>, Oskar Thorén <oskar@status.im>, Samuel Hawksby-Robinson <samuel@status.im> (alphabetical order)

## Abstract

This specification explains what Status account is, and how a node establishes trust.

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
- [Trust establishment](#trust-establishment)
    - [Terms Glossary](#terms-glossary)
    - [Contact Discovery](#contact-discovery)
        - [Public channels](#public-channels)
        - [Private 1:1 messages](#private-11-messages)
    - [Initial Key Exchange](#initial-key-exchange)
        - [Bundles](#bundles)
    - [Contact Verification](#contact-verification)
        - [Identicon](#identicon)
        - [3 word pseudonym / Whisper/Waku key fingerprint](#3-word-pseudonym--whisperwaku-key-fingerprint)
        - [ENS name](#ens-name)
- [Public Key Serialization](#public-key-serialization)
  - [Basic Serialization Example](#basic-serialization-example)
  - [Public Key "Compression" Rationale](#public-key-compression-rationale)
  - [Key Encoding](#key-encoding)
  - [Public Key Types](#public-key-types)
  - [De/Serialization Process Flow](#deserialization-process-flow)
    - [Serialization Example](#serialization-example)
    - [Deserialization Example](#deserialization-example)
- [Security Considerations](#security-considerations)
- [Changelog](#changelog)
  - [Version 0.3](#version-03)

<!-- markdown-toc end -->

## Introduction

The core concept of an account in Status is a set of cryptographic keypairs. Namely, the combination of the following:
1. a Whisper/Waku chat identity keypair
1. a set of cryptocurrency wallet keypairs

The node verifies or derives everything else associated with the contact from the above items, including:
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
- TODO: retrieval of long-time offline users bundle via `{IK}-contact-code` 

## Optional Account additions

### ENS Username
- A user MAY register a public username on the Ethereum Name System (ENS).  This username is a user-chosen subdomain of the `stateofus.eth` ENS registration that maps to their Whisper/Waku identity key (`IK`). 

<!-- ### User Profile Picture
- An account MAY edit the `IK` generated identicon with a chosen picture.  This picture will become part of the publicly broadcast profile of the account. -->

<!-- TODO: Elaborate on wallet account and multiaccount -->
<!-- TODO: Elaborate on security implications -->

## Trust establishment

**Trust establishment deals with users verifying they are communicating with who they think they are.**

### Terms Glossary

| term                      | description |
| ------------------------- | ----------- |
| privkey                   | ECDSA secp256k1 private key |
| pubkey                    | ECDSA secp256k1 public key |
| Whisper/Waku key          | pubkey for chat with HD derivation path m/43'/60'/1581'/0'/0 |


### Contact Discovery

#### Public channels
- Public group channels in Status are a broadcast/subscription system.  All public messages are encrypted with a symmetric key derived from the channel name, `K_{pub,sym}`, which is publicly known.
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
1. asynchronous X3DH key exchange
1. public key via public channel listening
    - `status-react/src/status_im/contact_code/core.cljs`
1. contact codes
1. decentralized storage (not implemented)
1. Whisper/Waku

### Initial Key Exchange

#### Bundles
- An X3DH prekey bundle is defined as ([code](https://github.com/status-im/status-go/messaging/chat/protobuf/encryption.pb.go)):
  ```
  Identity                // Identity key
  SignedPreKeys           // a map of installation id to array of signed prekeys by that installation id
  Signature               // Prekey signature
  Timestamp               // When the bundle was lasted created locally
  ```
  - include BundleContainer
- a new bundle SHOULD be created at least every 12 hours
- a node only generates a bundle when it is used
- a bundle SHOULD be distributed on the contact code channel. This is the Whisper and Waku topic `{IK}-contact-code`, where `IK` is the hex encoded public key of the user, prefixed with `0x`. The node encrypts the channel in the same way it encrypted public chats.

### Contact Verification

To verify that contact key information is as it should be, use the following.

#### Identicon
A low-poly identicon is deterministically generated from the Whisper/Waku chat public key.  This can be compared out of band to ensure the receiver's public key is the one stored locally.

#### 3 word pseudonym / Whisper/Waku key fingerprint
Status generates a deterministic 3-word random pseudonym from the Whisper/Waku chat public key.  This pseudonym acts as a human readable fingerprint to the Whisper/Waku chat public key.  This name also shows when viewing a contact's public profile and in the chat UI.
- implementation: [gfycat](https://github.com/status-im/status-react/tree/develop/src/status_im/utils/gfycat)

#### ENS name
Status offers the ability to register a mapping of a human readable subdomain of `stateofus.eth` to their Whisper/Waku chat public key. The user purchases this registration (currently by staking 10 SNT) and the node stores it on the Ethereum mainnet blockchain for public lookup.

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
- client - bootnode (go-ethereum)
    - a bootnode identifies itself by
        - an enode address
        - `NOTE: redezvous information here`
- client - ENS registry (ethereum blockchain -> default to infura)
- client - Ethereum RPC (custom go-ethereum RPC API -> default to infura API)
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

## Public Key Serialization

Idiomatically known as "public key compression" and "public key decompression".

The node SHOULD provide functionality for the serialization and deserialization of public / chat keys.

For maximum flexibility, when implementing this functionality, the node MUST support public keys encoded in a range of encoding formats, detailed below.

### Basic Serialization Example

In the example of a typical hexadecimal encoded elliptical curve (EC) public key (such as a secp256k1 pk),

```text
0x04261c55675e55ff25edb50b345cfb3a3f35f60712d251cbaaab97bd50054c6ebc3cd4e22200c68daf7493e1f8da6a190a68a671e2d3977809612424c7c3888bc6
```

minor modification for compatibility and flexibility makes the key self-identifiable and easily parsable, 

```text
fe70104261c55675e55ff25edb50b345cfb3a3f35f60712d251cbaaab97bd50054c6ebc3cd4e22200c68daf7493e1f8da6a190a68a671e2d3977809612424c7c3888bc6
```

EC serialization and compact encoding produces a much smaller string representation of the original key.

```text
zQ3shPyZJnxZK4Bwyx9QsaksNKDYTPmpwPvGSjMYVHoXHeEgB
```

### Public Key "Compression" Rationale

Serialized and compactly encoded ("compressed") public keys have a number of UI / UX advantages over non-serialized less densely encoded public keys.

Compressed public keys are smaller, and users may perceive them as less intimidating and less unnecessarily large. Compare the "compressed" and "uncompressed" version of the same public key from above example:

- `0xe70104261c55675e55ff25edb50b345cfb3a3f35f60712d251cbaaab97bd50054c6ebc3cd4e22200c68daf7493e1f8da6a190a68a671e2d3977809612424c7c3888bc6`
- `zQ3shPyZJnxZK4Bwyx9QsaksNKDYTPmpwPvGSjMYVHoXHeEgB`

The user can transmit and share the same data, but at one third of the original size. 136 characters uncompressed vs 49 characters compressed, giving a significant character length reduction of 64%. 

The user client app MAY use the compressed public keys throughout the user interface. For example in the `status-react` implementation of the user interface the following places could take advantage of a significantly smaller public key:  

- `Onboarding` > `Choose a chat name`
- `Profile` > `Header`
- `Profile` > `Share icon` > `QR code popover`
- `Invite friends` url from `Invite friends` button and `+ -button` > `Invite friends`
- Other user `Profile details` 
- `Profile details` > `Share icon` > `QR code popover`

In the case of QR codes a compressed public key can reduce the complexity of the derived codes:

| Uncompressed | Compressed |
| --- | --- |
|<img src="https://user-images.githubusercontent.com/5702426/80531063-e98fcc80-8991-11ea-8c02-c354b5828d35.png" width="400" />|<img src="https://user-images.githubusercontent.com/5702426/80501933-f4356c00-8967-11ea-87d8-eae18becece9.png" width="400"/>|


### Key Encoding

When implementing the pk de/serialization functionality, the node MUST use the [multiformats/multibase](https://github.com/multiformats/multibase) encoding protocol to interpret incoming key data and to return key data in a desired encoding.

The node SHOULD support the following `multibase` encoding formats.

```csv
encoding,          code, description,                                              status
identity,          0x00, 8-bit binary (encoder and decoder keeps data unmodified), default
base2,             0,    binary (01010101),                                        candidate
base8,             7,    octal,                                                    draft
base10,            9,    decimal,                                                  draft
base16,            f,    hexadecimal,                                              default
base16upper,       F,    hexadecimal,                                              default
base32hex,         v,    rfc4648 case-insensitive - no padding - highest char,     candidate
base32hexupper,    V,    rfc4648 case-insensitive - no padding - highest char,     candidate
base32hexpad,      t,    rfc4648 case-insensitive - with padding,                  candidate
base32hexpadupper, T,    rfc4648 case-insensitive - with padding,                  candidate
base32,            b,    rfc4648 case-insensitive - no padding,                    default
base32upper,       B,    rfc4648 case-insensitive - no padding,                    default
base32pad,         c,    rfc4648 case-insensitive - with padding,                  candidate
base32padupper,    C,    rfc4648 case-insensitive - with padding,                  candidate
base32z,           h,    z-base-32 (used by Tahoe-LAFS),                           draft
base36,            k,    base36 [0-9a-z] case-insensitive - no padding,            draft
base36upper,       K,    base36 [0-9a-z] case-insensitive - no padding,            draft
base58btc,         z,    base58 bitcoin,                                           default
base58flickr,      Z,    base58 flicker,                                           candidate
base64,            m,    rfc4648 no padding,                                       default
base64pad,         M,    rfc4648 with padding - MIME encoding,                     candidate
base64url,         u,    rfc4648 no padding,                                       default
base64urlpad,      U,    rfc4648 with padding,                                     default
```

**Note** this specification RECOMMENDs that implementations extend the standard `multibase` protocol to parse strings prepended with `0x` as `f` hexadecimal encoded bytes.

Implementing this recommendation will allow the node to correctly interpret traditionally identified hexadecimal strings (e.g. `0x1337c0de`).

*Example:*

`0xe70102261c55675e55ff25edb50b345cfb3a3f35f60712d251cbaaab97bd50054c6ebc`

SHOULD be interpreted as 

`fe70102261c55675e55ff25edb50b345cfb3a3f35f60712d251cbaaab97bd50054c6ebc`

This specification RECOMMENDs that the consuming service of the node uses a compact encoding type, such as base64 or base58 to allow for as short representations of the key as possible.

### Public Key Types

When implementing the pk de/serialization functionality, The node MUST support the [multiformats/multicodec](https://github.com/multiformats/multicodec) key type identifiers for the following public key type.

| Name               | Tag | Code   | Description                          |
| ------------------ | --- | ------ | ------------------------------------ |
| `secp256k1-pub`    | key | `0xe7` | Secp256k1 public key                 |

For a public key to be identifiable to the node the public key data MUST be prepended with the relevant [multiformats/unsigned-varint](https://github.com/multiformats/unsigned-varint) formatted code.

*Example:*

Below is a representation of an deserialized secp256k1 public key.

```text
04
26 | 1c | 55 | 67 | 5e | 55 | ff | 25
ed | b5 | 0b | 34 | 5c | fb | 3a | 3f
35 | f6 | 07 | 12 | d2 | 51 | cb | aa
ab | 97 | bd | 50 | 05 | 4c | 6e | bc
3c | d4 | e2 | 22 | 00 | c6 | 8d | af
74 | 93 | e1 | f8 | da | 6a | 19 | 0a
68 | a6 | 71 | e2 | d3 | 97 | 78 | 09
61 | 24 | 24 | c7 | c3 | 88 | 8b | c6
```

The `multicodec` code for a secp256k1 public key is `0xe7`.

After parsing the code `0xe7` as a `multiformats/uvarint`, the byte value is `0xe7 0x01`, prepending this to the public key results in the below representation.

```text
e7 | 01 | 04
26 | 1c | 55 | 67 | 5e | 55 | ff | 25
ed | b5 | 0b | 34 | 5c | fb | 3a | 3f
35 | f6 | 07 | 12 | d2 | 51 | cb | aa
ab | 97 | bd | 50 | 05 | 4c | 6e | bc
3c | d4 | e2 | 22 | 00 | c6 | 8d | af
74 | 93 | e1 | f8 | da | 6a | 19 | 0a
68 | a6 | 71 | e2 | d3 | 97 | 78 | 09
61 | 24 | 24 | c7 | c3 | 88 | 8b | c6
```

### De/Serialization Process Flow

When implementing the pk de/serialization functionality, the node MUST be passed a `multicodec` identified public key, of the above supported types, encoded with a valid `multibase` identifier.

This specification RECOMMENDs that the node also accept an encoding type parameter to encode the output data. This provides for the case where the user requires the de/serialization key to be in a different encoding to the encoding of the given key. 

#### Serialization Example

A hexadecimal encoded secp256k1 public chat key typically is represented as below:

```text
0x04261c55675e55ff25edb50b345cfb3a3f35f60712d251cbaaab97bd50054c6ebc3cd4e22200c68daf7493e1f8da6a190a68a671e2d3977809612424c7c3888bc6
``` 

To be properly interpreted by the node for serialization the public key MUST be prepended with the `multicodec` `uvarint` code `0xea 0x01` and encoded with a valid `multibase` encoding, therefore giving the following:

```text
fea0104261c55675e55ff25edb50b345cfb3a3f35f60712d251cbaaab97bd50054c6ebc3cd4e22200c68daf7493e1f8da6a190a68a671e2d3977809612424c7c3888bc6
``` 

If adhering to the specification recommendation to provide the user with an output encoding parameter, the above string would be passed to the node with the following `multibase` encoding identifier.

In this example the output encoding is defined as `base58 bitcoin`.

```text
z
``` 

The return value in this case would be

```text
zQ3shPyZJnxZK4Bwyx9QsaksNKDYTPmpwPvGSjMYVHoXHeEgB
```

Which after `multibase` decoding can be represented in bytes as below:

```text
e7 | 01 | 02
26 | 1c | 55 | 67 | 5e | 55 | ff | 25
ed | b5 | 0b | 34 | 5c | fb | 3a | 3f
35 | f6 | 07 | 12 | d2 | 51 | cb | aa
ab | 97 | bd | 50 | 05 | 4c | 6e | bc
```

#### Deserialization Example

For the user, the deserialization process is exactly the same as serialization with the exception that the user MUST provide a serialized public key for deserialization. Else the deserialization algorithm will fail.

For further guidance on the implementation of public key de/serialization consult the [`status-go` implementation and tests](https://github.com/status-im/status-go/blob/c9772325f2dca76b3504191c53313663ca2efbe5/api/utils_test.go).  

## Security Considerations

-

## Changelog

### Version 0.4

Released // TODO

- Added details of public key serialization and deserialization

### Version 0.3

Released [May 22, 2020](https://github.com/status-im/specs/commit/664dd1c9df6ad409e4c007fefc8c8945b8d324e8)

- Added language to include Waku in all relevant places
- Change to keep `Mailserver` term consistent 
- Added clarification to Open Whisper Systems
