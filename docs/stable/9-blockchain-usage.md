---
permalink: /spec/9
parent: Stable specs
title: 9/BLOCKCHAIN-USAGE
---

# 3/BLOCKCHAIN-USAGE

> Version: 0.1
>
> Status: Stable
>
> Authors: Andrea Maria Piana <andreap@status.im>

# Status interactions with the blockchain

In this document is document all the interactions that the Status client has
with the blockchain.

1. [Wallet](#Wallet)
2. [ENS](#ENS)

## Wallet

The wallet in Status has two main components:

1) Sending transactions
2) Fetching balance

In the section below are described the `RPC` calls made the nodes, with a brief
description of their functionality and how it is used by Status.

1. [Sending transactions](#Sending-transactions)
  *[EstimateGas](#EstimateGas)
  *[PendingNonceAt](#PendingNonceAt)
  *[SuggestGasPrice](#SuggestGasPrice)
  *[SendTransaction](#SendTransaction)
2. [Fetching balance](#Fetching-balance)
  *[BlockByHash](#BlockByHash)
  *[BlockByNumber](#BlockByNumber)
  *[FilterLogs](#FilterLogs)
  *[HeaderByNumber](#HeaderByNumber)
  *[NonceAt](#NonceAt)
  *[TransactionByHash](#TransactionByHash)
  *[TransactionReceipt](#TransactionReceipt)

### Sending transactions

#### EstimateGase

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

#### BlockByHash

BlockByHash returns the given full block.

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
