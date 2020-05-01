---
title: IPFS gateway for Sticker Pack
version: 0.1.0
status: Draft
authors: Gheorghe Pinzaru <gheorghe@status.im>
redirect_from:
  - /sticker-pack.html
---

## Table of Contents

 1. [Abstract](#abstract)
 2. [Specification](#specification)
 3. [Copyright](#copyright)

## Abstract

In this specification, we describe how Status app uses IPFS gateway to store stickers.
We will explore image format, how they are uploaded and how an end user can see them
insed the Status app.

## Definition

| Term             | Description                                                                            |
|------------------|----------------------------------------------------------------------------------------|
| **Stickers**     | A set of images which can be used to express emotions                                  |
| **Sticker Pack** | ERC721 token which includes the set of stickers                                        |
| **IPFS**        | P2P network used to store and share data, in this case, the images for the stickerpack |

## Specification

### Image format
Accepted image file types are **PNG, JPG/JPEG and GIF**, with a maximum allowed size of 300kb.
The minimum sticker image resoulution is 512x512, and background of clipart should be transparent.

### Distribution

Sticker pack are implemented as ERC721 token and contains a set of stickers. These stickers
are stored inside the sticker pack as a set of hyperlinks pointing to IPFS storage. These
hyperlinks are publically available and can be accessed by any user inside the status chat.
Stickers can be sent in chat only by accounts that own the sticker pack.

### IPFS gateway
At the moment of writing, the Status uses the [Infura](https://infura.io/) gateway.
Infura gateway is an HTTPS gateway, which based on an HTTP GET request with the
multihash block will return the stored content at that block address. 

The use of gateway is required to enable easy access to the resources over HTTP.
Each image of a sticker is stored inside IPFS using a unique address that is 
derived from the hash of the file. This ensures that a file can't be overridden,
and an end-user of the IPFS will receive the same file at a given address.

The IPFS gateway acts as an end-user of the IPFS and allows users of the gateway
to access IPFS without connection to the P2P network. Usage of a gateway introduces
potential risk for the users of that gateway provider. In case of a compromising
of security of the provider, meta information of its users can be leaked. Or if the
provider servers are unavailable the access to the whole IPFS network will be lost.

When a sticker is shown in the app, Status app makes an http GET request to IPFS gateway using the hyperlink. 

### Status sticker usage
To send a sticker in chat, a user of Status should  buy or install a sticker pack.
To install a sticker pack, we need to fetch all sticker packs which are available in Sticker Market. 
The following steps are needed to fetch all sticker packs:
#### 1. Get total number of sticker packs
Call `packCount()` on the sticker market contract, will return number of sticker pack registered as `uint256`.
#### 2. Get sticker pack by id
ID's are represented as `uint256` and are incremental from `0` to totatl number of sticker packs in contract, which we received on previous step. To get a sticker pack we should call `getPackData(sticker-pack-id)`, the return type is  `["bytes4[]" "address" "bool" "uint256" "uint256" "bytes"]` which represents the following fields: `[category owner mintable timestamp price contenthash]`. 
`TODO:` Describe contenthash, it's a hash in IPFS (what it contains?)
##### 3. Get owned stick packs
To get owned packs, we should get all owned tokens for the current account address. To do that we should call `balanceOf(address)` where address is the address for current account. This method returns a `uint256` representing the token id representing the count of available tokens. Using `tokenOfOwnerByIndex(address,uint256)` method, with the address of the user and ID in form of a `uint256` which is an incremented int from 0 to total number of tokens, we will get token id. To get sticker pack id from token we call`tokenPackId(uint256)` where `uint256` is the token id. This method will return an `uint256` which is the id of the owned sticker pack.

##### 4. Buy a sticker pack
To buy a sticker pack we should call `approveAndCall(address,uint256,bytes)` where `address` is the address of buyer,`uint256` is the price and third parameters `bytes` is the callback  called if approved. In callback we call `buyToken(uint256,address,uint256)`, first parameter is sticker pack id, second buyers address, and the last is the price.
## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

