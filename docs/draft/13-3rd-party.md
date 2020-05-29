---
permalink: /spec/13
parent: Draft specs
title: 13/3RD-PARTY-USAGE
---

# 13/3RD-PARTY

> Version: 0.1
>
> Status: Draft
>
> Authors: Volodymyr Kozieiev <volodymyr@status.im>


# Third party APIs used for core functionality

## Table of Contents

1. [Abstract](#abstract)
2. [Definitions](#definitions)
3. [Why 3rd party API can be a problem?](#why-3rd-party-api-can-be-a-problem)
4. [3rd party APIs used by Status](#3rd-party-apis-used-by-current-status-app)
  * [Infura](#infura)
  * [Etherscan](#etherscan)
  * [CryptoCompare](#cryptocompare)
  * [Collectibles](#collectibles)
  * [Iubenda](#iubenda)
5. [Changelog](#changelog)
6. [Copyright](#copyright)

## Abstract

This specification discusses 3rd party APIs that Status relies on. These APIs provide various capabilities such as:
- communicate with the Ethereum network
- allow users to see address and transaction details on external website
- get fiat/crypto exchange rates
- get information about collectibles
- hosts privacy policy

## Definitions

| Term        | Description |
| ------------- |-------------|
| Fiat money    | Currency which established as money, often by government regulation, but that has no intrinsic value
| Full node    | Any computer, connected to the Ethereum network, which fully enforces all the consensus rules of Ethereum.
| Crypto-collectible | A cryptographically unique, non-fungible digital asset . Unlike cryptocurrencies, which require all tokens to be identical, each crypto-collectible token is unique or limited in quantity.


## Why 3rd party API can be a problem?
Relying on 3rd party APIs interferes with `censorship resistance` Status principle. Since Status aims to avoid suppression of information it is important to reduce amount of 3rd parties crucial for app functionality.

## 3rd party APIs used by current Status app

### Infura

##### What is it?
Infura hosts a collection of full nodes for the Ethereum network and provides an API to access both the Ethereum and IPFS networks without having to run a full node.

##### How Status use it?
Status works on mobile devices and therefore can't rely on local node. So all communication to Ethereum network happens via Infura.

##### Concerns
Making http request means that a user leaks metadata, which can be used in various attacks if the service is hacked.
Infura hosts on centralized providers. If these fail or the provider cuts off service, then Status features requiring Ethereum calls will.


### Etherscan
##### What is it?
Etherscan is a service that allows user to explore and search the Ethereum blockchain for transactions, addresses, tokens, prices and other activities taking place on Ethereum.

##### How Status use it?
Status Wallet allows users to view details of addresses and transactions on Etherscan.

##### Concerns
If Etherscan fails user won't be able to view address or transaction details with it. But inside the app this info will still be available.

### CryptoCompare

##### What is it?
CryptoCompare is a service that shows live streaming prices, charts and analysis from top crypto exchanges.

##### How Status use it?
Status regularly fetches crypto prices from CryptoCompare. Using that info Status calculates fiat value for transaction or wallet assets.

##### Concerns
Making http request means that a user leaks metadata, which can be used in various attacks if the service is hacked.
If CryptoCompare fails Status won't be able to show fiat equivalent of crypto in wallet.

### Collectibles

There is a set of services that used for getting information about collectibles:
- https://api.pixura.io/graphql
- https://www.etheremon.com/api
- https://us-central1-cryptostrikers-prod.cloudfunctions.net/cards/
- https://api.cryptokitties.co/


##### Concerns
Making http request means that a user leaks metadata, which can be used in various attacks if the service is hacked.

### Iubenda

##### What is it?

Service that helps in creating documents that make websites and apps compliant with the law across multiple countries and legislations.

##### How Status use it?
Privacy policy of Status hosted on Iubenda.

##### Concerns
If Iubenda fails Status users won't be able to navigate to app's privacy policy.

## Changelog

| Version | Comment |
| :-----: | ------- |
| [0.1.0](https://github.com/status-im/specs/blob/master/docs/draft/9-3rd-party.md)   | Initial Release |

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
