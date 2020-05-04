---
title: 3rd party APIs used for core functionality that impacts things like availability/censorship and privacy
version: 0.1.0
status: Draft
authors:
---

# 3rd party APIs used for core functionality that impacts things like availability/censorship and privacy

## Table of Contents

1. [Abstract](Abstract)
2. [Definitions](#definitions)
3. [Why 3rd party API can be a problem?](#why-3rd-party-api-can-be-a-problem)
4. [3rd party APIs used by Status](#3rd-party-apis-used-by-status)
  * [Infura](#infura)
  * [Etherscan](#etherscan)
  * [CryptoCompare](#cryptocompare)
  * [Collectibles](#collectibles)
  * [Iubenda](#iubenda)
5. [Changelog](#changelog)
6. [Acknowledgements](#acknowledgements)
7. [Copyright](#copyright)

## Abstract
In this specification listed 3rd party APIs that Status functionality rely on.

## Definitions

| Term        | Description |
| ------------- |-------------|
| Fiat money    | Currency which established as money, often by government regulation, but that has no intrinsic value
| Full node    | Any computer, connected to the Ethereum network, which fully enforces all the consensus rules of Ethereum.
| Crypto-collectible | A cryptographically unique, non-fungible digital asset . Unlike cryptocurrencies, which require all tokens to be identical, each crypto-collectible token is unique or limited in quantity.


## Why 3rd party API can be a problem?
Relying on 3rd party APIs interferes with `censorship resistance` Status principle. Since we aim to avoid suppression of information it is important to reduce amount of 3rd parties crucial for app functionality.

## 3rd party APIs used by Status

### Infura

##### What is it?
Infura hosts a collection of own full nodes on the Ethereum network and provides an API access to the Ethereum and IPFS networks without having to run a full node.

##### How Status use it?
Status works on mobile devices and therefore can't rely on local node. So all communication to Ethereum network happens via Infura.

##### Concerns
Making http request means that user metadata leaks. Also if service hacked it can be used in various attacks, e.g. by faking returning data.
Infura hosts on Amazon. It can fail or Amazon can cut off service or their servers crash. In this case all Status features related to Ethereum network calls will fail.


### Etherscan
##### What is it?
Etherscan is a service that allows user to explore and search the Ethereum blockchain for transactions, addresses, tokens, prices and other activities taking place on Ethereum network.

##### How Status use it?
Status Wallet has buttons that allow user to view details of address or transactions on Etherscan site.

##### Concerns
If Etherscan fails user won't be able to view address or transaction details with it. But inside the app this info will still be available.

### CryptoCompare

##### What is it?
CryptoCompare is a service that shows live streaming prices, charts and analysis from top crypto exchanges.

##### How Status use it?
Status regularly fetches crypto prices from CryptoCompare. Using that info Status calculates fiat value for transaction or wallet assets.

##### Concerns
Making http request means that user metadata leaks. Also if service hacked it can be used in various attacks, e.g. by faking returning data.
If CryptoCompare fails Status won't be able to show fiat equivalent of crypto in wallet.

### Collectibles

There is a set of services that used for getting information about collectibles:
- https://api.pixura.io/graphql
- https://www.etheremon.com/api
- https://us-central1-cryptostrikers-prod.cloudfunctions.net/cards/
- https://api.cryptokitties.co/


##### Concerns
Making http request means that user metadata leaks. Also if service hacked they can be used in various attacks, e.g. by faking returning data.

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
| [0.1.0](https://github.com/specs/...)   | Initial Release |

## Acknowledgements

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
