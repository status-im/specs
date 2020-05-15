---
permalink: /spec/8
parent: Draft specs
title: 8/IPFS gateway for Sticker Pack
---

# 8/IPFS gateway for Sticker Pack

> Version: 0.1.0
>
> Status: Draft
>
> Authors: Gheorghe Pinzaru <gheorghe@status.im>
>


## Table of Contents

 1. [Abstract](#abstract)
 2. [Specification](#specification)
 3. [Copyright](#copyright)

## Abstract

In this specification, we describe how Status uses the IPFS gateway to store stickers.
We will explore image format, how they are uploaded and how an end user can see them inside the Status app.

## Definition

| Term             | Description                                                                            |
|------------------|----------------------------------------------------------------------------------------|
| **Stickers**     | A set of images which can be used to express emotions                                  |
| **Sticker Pack** | ERC721 token which includes the set of stickers                                        |
| **IPFS**         | P2P network used to store and share data, in this case, the images for the stickerpack |

## Specification

### Image format
Accepted image file types are `PNG`, `JPG/JPEG` and `GIF`, with a maximum allowed size of 300kb.
The minimum sticker image resolution is 512x512, and its background SHOULD be transparent.

### Distribution

Sticker packs are implemented as [ERC721 token](https://eips.ethereum.org/EIPS/eip-721) and contain a set of stickers. These stickers
are stored inside the sticker pack as a set of hyperlinks pointing to IPFS storage. These hyperlinks are publically available and can be accessed by any user inside the status chat.
Stickers can be sent in chat only by accounts that own the sticker pack.

### IPFS gateway
At the moment of writing, the current main Status app uses the [Infura](https://infura.io/) gateway. However clients could choose a different gateway or to run own IPFS node.
Infura gateway is an HTTPS gateway, which based on an HTTP GET request with the multihash block will return the stored content at that block address. 

The use of gateway is required to enable easy access to the resources over HTTP.
Each image of a sticker is stored inside IPFS using a unique address that is 
derived from the hash of the file. This ensures that a file can't be overridden, and an end-user of the IPFS will receive the same file at a given address.

### Security
The IPFS gateway acts as an end-user of the IPFS and allows users of the gateway to access IPFS without connection to the P2P network.
Usage of a gateway introduces potential risk for the users of that gateway provider. In case of a compromise in the security of the provider, meta information such as IP address, User-Agent and other of its users can be leaked.
If the provider servers are unavailable the access trought gateway to the IPFS network is lost.

### Status sticker usage
When a sticker is shown in the app, Status app makes an http GET request to IPFS gateway using the hyperlink. 

To send a sticker in chat, a user of Status should buy or install a sticker pack.

To be available for installation a Sticker Pack should be submitted to Sticker market by an author.

#### Submit a sticker

To submit a sticker pack, the author should upload all assets to IPFS. Then generate a payload including name, author, thumbnail, preview and a list of stickers in the [EDN format](https://github.com/edn-format/edn). Following this structure:
```
{meta {:name "Sticker pack name"
       :author "Author Name"
       :thumbnail "e30101701220602163b4f56c747333f43775fdcbe4e62d6a3e147b22aaf6097ce0143a6b2373"
       :preview "e30101701220ef54a5354b78ef82e542bd468f58804de71c8ec268da7968a1422909357f2456"
       :stickers [{:hash "e301017012207737b75367b8068e5bdd027d7b71a25138c83e155d1f0c9bc5c48ff158724495"}
       {:hash "e301017012201a9cdea03f27cda1aede7315f79579e160c7b2b6a2eb51a66e47a96f47fe5284"}]}}
```
All assets fileds, are contenthash fileds as per [EIP 1577](https://eips.ethereum.org/EIPS/eip-1577).
 This payload is uploaded also to IPFS, and the IPFS address is used in the content field of the Sticker Market contract. See [Sticker Market spec](https://github.com/status-im/sticker-market/blob/651e88e5f38c690e57ecaad47f46b9641b8b1e27/docs/specification.md) for a detailed description of the contract.

#### Install a sticker pack

To install a sticker pack, we need to fetch all sticker packs which are available in Sticker Market. The following steps are needed to fetch all sticker packs:

#### 1. Get total number of sticker packs
Call `packCount()` on the sticker market contract, will return number of sticker pack registered as `uint256`.

#### 2. Get sticker pack by id
ID's are represented as `uint256` and are incremental from `0` to total number of sticker packs in contract, which we received on previous step. To get a sticker pack we should call `getPackData(sticker-pack-id)`, the return type is  `["bytes4[]" "address" "bool" "uint256" "uint256" "bytes"]` which represents the following fields: `[category owner mintable timestamp price contenthash]`. Price is the SNT value in wei setted by sticker pack owner. The contenthash is the IPFS address described in the [submit description](#submit-a-sticker-pack) above. Other fields specification could be found in [Sticker Market spec](https://github.com/status-im/sticker-market/blob/651e88e5f38c690e57ecaad47f46b9641b8b1e27/docs/specification.md)

##### 3. Get owned sticker packs
The current Status app fetches owned sticker packs during the open of any sticker view (a screen which shows a sticker pack or the list of sticker packs).
To get owned packs, we should get all owned tokens for the current account address. To do that we should call `balanceOf(address)` where address is the address for current account. This method returns a `uint256` representing the count of available tokens. Using `tokenOfOwnerByIndex(address,uint256)` method, with the address of the user and ID in form of a `uint256` which is an incremented int from 0 to total number of tokens, we will get token id. To get sticker pack id from token we call`tokenPackId(uint256)` where `uint256` is the token id. This method will return an `uint256` which is the id of the owned sticker pack.

##### 4. Buy a sticker pack
To buy a sticker pack we should call `approveAndCall(address,uint256,bytes)` where `address` is the address of buyer,`uint256` is the price and third parameters `bytes` is the callback  called if approved. In callback we call `buyToken(uint256,address,uint256)`, first parameter is sticker pack id, second buyers address, and the last is the price.

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
