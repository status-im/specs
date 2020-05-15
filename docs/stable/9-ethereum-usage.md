---
permalink: /spec/9
parent: Stable specs
title: 9/ETHEREUM-USAGE
---

# 9/ETHEREUM-USAGE

> Version: 0.1
>
> Status: Stable
>
> Authors: Andrea Maria Piana <andreap@status.im>

# Status interactions with the Ethereum blockchain

In this document we document all the interactions that the Status client has
with the [Ethereum](https://ethereum.org/developers/) blockchain.

All the interactions are made through [JSON-RPC](https://github.com/ethereum/wiki/wiki/JSON-RPC).
Currently [Infura](https://infura.io/) is used. The client assumes high-availability, otherwise 
it will not be able to interact with the Ethereum blockchain.
We rely on these nodes to validate the integrity of the transaction and report a 
consistent history.

Key handling is described [here](./2-account.md)

1. [Wallet](#Wallet)
2. [ENS](#ENS)

## Wallet

The wallet in Status has two main components:

1) Sending transactions
2) Fetching balance

In the section below are described the `RPC` calls made the nodes, with a brief
description of their functionality and how it is used by Status.

1. [Sending transactions](#Sending-transactions)
  - [EstimateGas](#EstimateGas)
  - [PendingNonceAt](#PendingNonceAt)
  - [SuggestGasPrice](#SuggestGasPrice)
  - [SendTransaction](#SendTransaction)
2. [Fetching balance](#Fetching-balance)
  - [BlockByHash](#BlockByHash)
  - [BlockByNumber](#BlockByNumber)
  - [FilterLogs](#FilterLogs)
  - [HeaderByNumber](#HeaderByNumber)
  - [NonceAt](#NonceAt)
  - [TransactionByHash](#TransactionByHash)
  - [TransactionReceipt](#TransactionReceipt)

### Sending transactions

#### EstimateGas

EstimateGas tries to estimate the gas needed to execute a specific transaction based on
the current pending state of the backend blockchain. There is no guarantee that this is
the true gas limit requirement as other transactions may be added or removed by miners,
but it should provide a basis for setting a reasonable default.

```
func (ec *Client) EstimateGas(ctx context.Context, msg ethereum.CallMsg) (uint64, error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L499


#### PendingNonceAt
`PendingNonceAt` returns the account nonce of the given account in the pending state.
 This is the nonce that should be used for the next transaction.

 ```
func (ec *Client) PendingNonceAt(ctx context.Context, account common.Address) (uint64, error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L440


#### SuggestGasPrice

`SuggestGasPrice` retrieves the currently suggested gas price to allow a timely
execution of a transaction.

```
func (ec *Client) SuggestGasPrice(ctx context.Context) (*big.Int, error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L487

#### SendTransaction

`SendTransaction` injects a signed transaction into the pending pool for execution.

If the transaction was a contract creation use the TransactionReceipt method to get the
contract address after the transaction has been mined.


```
func (ec *Client) SendTransaction(ctx context.Context, tx *types.Transaction) error 
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L512

### Fetching balance

We currently fetch the current and historical [ECR20] (https://eips.ethereum.org/EIPS/eip-20) and ETH balance for the user wallet address.
Collectibles following the [ECR-721](https://eips.ethereum.org/EIPS/eip-721) are also fetched if enabled.

We support by default the following [tokens](https://github.com/status-im/status-react/blob/develop/src/status_im/ethereum/tokens.cljs). Custom tokens can be added by specifying the `address`, `symbol` and `decimals`.

#### BlockByHash

`BlockByHash` returns the given full block.

It is used by status to fetch a given block which will then be inspected for 
transfers to the user address, both tokens and ETH.

```
func (ec *Client) BlockByHash(ctx context.Context, hash common.Hash) (*types.Block, error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L78

#### BlockByNumber

`BlockByNumber` returns a block from the current canonical chain. If number is nil, the
latest known block is returned.

```
func (ec *Client) BlockByNumber(ctx context.Context, number *big.Int) (*types.Block, error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L82

#### FilterLogs

`FilterLogs` executes a filter query.

Status uses this function to filter out logs, using the hash of the block
and the address that we are interested in, both inbound and outbound.

```
func (ec *Client) FilterLogs(ctx context.Context, q ethereum.FilterQuery) ([]types.Log, error) 
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L377


#### NonceAt

`NonceAt` returns the account nonce of the given account.

```
func (ec *Client) NonceAt(ctx context.Context, account common.Address, blockNumber *big.Int) (uint64, error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L366


#### TransactionByHash

`TransactionByHash` returns the transaction with the given hash, used to inspect those 
transaction made/received by the user.

```
func (ec *Client) TransactionByHash(ctx context.Context, hash common.Hash) (tx *types.Transaction, isPending bool, err error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L202

#### HeaderByNumber

`HeaderByNumber` returns a block header from the current canonical chain.

```
func (ec *Client) HeaderByNumber(ctx context.Context, number *big.Int) (*types.Header, error) 
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L172


#### TransactionReceipt

`TransactionReceipt` returns the receipt of a transaction by transaction hash.
It is used in status to check if a token transfer was made to the user address.

```
func (ec *Client) TransactionReceipt(ctx context.Context, txHash common.Hash) (*types.Receipt, error)
```

https://github.com/ethereum/go-ethereum/blob/26d271dfbba1367326dec38068f9df828d462c61/ethclient/ethclient.go#L270


## ENS

All the interactions with `ENS` are made through the [ENS contract](https://github.com/ensdomains/ens)

For the `stateofus.eth` username, one can be registered through these [contracts](https://github.com/status-im/ens-usernames)

### Registering, releasing and updating

- [Registering a username](https://github.com/status-im/ens-usernames/blob/77d9394d21a5b6213902473b7a16d62a41d9cd09/contracts/registry/UsernameRegistrar.sol#L113)
- [Releasing a username](https://github.com/status-im/ens-usernames/blob/77d9394d21a5b6213902473b7a16d62a41d9cd09/contracts/registry/UsernameRegistrar.sol#L131)
- [Updating a username] (https://github.com/status-im/ens-usernames/blob/77d9394d21a5b6213902473b7a16d62a41d9cd09/contracts/registry/UsernameRegistrar.sol#L174)

### Slashing

Usernames MUST be in a specific format, otherwise they MAY be slashed:

- They MUST only contain alphanumeric characters
- They MUST NOT  be in the form `0x[0-9a-f]{5}.*` and have more than 12 characters
- They MUST NOT be in the [reserved list] (https://github.com/status-im/ens-usernames/blob/47c4c6c2058be0d80b7d678e611e166659414a3b/config/ens-usernames/reservedNames.js)
- They MUST NOT be too short, this is dinamically set in the contract and can be checked against the [contract](https://github.com/status-im/ens-usernames/blob/master/contracts/registry/UsernameRegistrar.sol#L26)

- [Slash a reserved username](https://github.com/status-im/ens-usernames/blob/77d9394d21a5b6213902473b7a16d62a41d9cd09/contracts/registry/UsernameRegistrar.sol#L237)
- [Slash an invalid username] (https://github.com/status-im/ens-usernames/blob/77d9394d21a5b6213902473b7a16d62a41d9cd09/contracts/registry/UsernameRegistrar.sol#L261)
- [Slash a username too similar to an address](https://github.com/status-im/ens-usernames/blob/77d9394d21a5b6213902473b7a16d62a41d9cd09/contracts/registry/UsernameRegistrar.sol#L215)
- [Slash a username that is too short](https://github.com/status-im/ens-usernames/blob/77d9394d21a5b6213902473b7a16d62a41d9cd09/contracts/registry/UsernameRegistrar.sol#L200)

ENS names are propagated through `ChatMessage` and `ContactUpdate` [payload](./6-payloads.md).
A client SHOULD verify ens names against the public key of the sender on receiving the message against the [ENS contract](https://github.com/ensdomains/ens)

## Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

