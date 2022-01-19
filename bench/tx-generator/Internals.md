# tx-generator Service and Maintenance Manual
This document describes some key concepts of the tx-generator
and is a good starting point for anybody who wants to fix bugs
or add new features to the tx-generator.

## Benchmarking Scripts
The transaction generator is basically an interpreter for
benchmarking scripts.
A script defines how transactions are generated and transmitted.
Scripts can also query the state of the blockchain
to wait for era transitions
and it is also possible to write scripts that never interact
with Cardano-node, but instead write all the generated transactions to a file.
Benchmarking scripts (or just scripts) should not be confused with Plutus scripts.

## Main
The start-up of the tx-generator is pretty simple:

### `Cardano.Benchmarking.Command.runCommand`

* Call `withIOManager`.
* Parse the CLI arguments.
* Read and parse the benchmarking script.
* Call `Cardano.Benchmarking.Script.runScript`.
* Catch some errors.

In the executable `main = runCommand`.

### `Cardano.Benchmarking.Script.runScript`
* Setup the `ActionM` Monad.
* Initialize some rudimentary Tracing/Logging.
* Initialize the globalWallet.
* Interpret the benchmarking Script: `forM_ script action`
* Catch error and shutdown the Logging layer.

Note:
Many (but not all) scenarios of using the tx-generator involve additional setup actions.
For example: Reading the protocol parameters, reading a node-config,
starting a full cardano-node compatible logging infrastructure, etc.
These extra setup steps are provided by explicit script commands at the top
of the script.

Scripts that do not make use of node-to-node connections do not need
to run some of the setup step.
Examples are scripts for benchmarking the tx-generator itself,
or smoke tests, that write the generated txs to a file instead of transmitting
them to a node.

### `Cardano.Benchmarking.Script.Action.action :: Action -> ActionM ()`
This is a big case-dispatch which calls the implementation of each action.

### `module Cardano.Benchmarking.Script.Core`
This module is a collection of reusable building blocks for benchmarking scripts
and the implementations for all the script actions.
Most function here run in the `ActionM` monad.

## Benchmarking Scripts , State and the  `ActionM` Monad

Scripts send transactions to the blockchain and spend UTxOs.
Therefore they are inherently stateful.
The tx-generator maintains internal state (environment)
and tries to keep track of the set of UTxOs that are available for the tx-generator.

The internal state of the tx-generator is available via the `ActionM` Monad.
This includes `MVar`s which contain IO-update-able state.
The state/environment is defined in `Cardano.Benchmarking.Script.Env`.

## Wallets and Funds
The transaction generator has an internal notion of wallets and funds.
Wallets store a set of Funds.
A Fund is basically an UTxO together with
all the information that is needed for spending,
i.e. keys, Plutus scripts and Plutus redeemers, etc.

Note:
The wallet implementation uses `ixset-typed.
Unfortunately, it turned out that
queries have a poor complexity in `ixset-txped`.
More precisely, a query seems to be linear in the size of data set.
(In practice, it becomes notable at about 10_000 stored funds
and takes minutes for data sets of size  1_000_000.)
The computational costs is the same regardless
whether one extracts one fund from the query or all the matching funds.

The tx-generator uses the following workaround to speed up the queries.
Instead of running an `ixset-typed` query for each fund/UTxO,
it queries all the funds that are needed (for a particular phase)
in on query and buffers the funds in a plain old list.

This workaround works well in the current state of the tx-generator.
`ixset` based wallets provide a nice abstraction for future
extension to the tx-generator, but one has to keep in mind
the complexities of the data structures.

## Lifetime of a Transaction
Benchmarking scripts can create a variety of transactions for different purposes:

* Splitting funds.
* Regular transactions.
* Paying to a Plutus script.
* Spending a script address.

Under the hood, all transactions are generated in the same function
and with the same workflow.
This is done in `Cardano.Benchmarking.Wallet.sourceToStoreTransaction`.
It performs the following steps:

* Select the input Funds:
  `type FundSource = IO (Either String [Fund])`
* Determin output values (value preserving) :
  `[Lovelace] -> [Lovelace]`
* Create UTxOs for the ouput values:
  `type ToUTxO era = [Lovelace] -> ([TxOut CtxTx era], TxId -> [Fund])`
* Create the actual transaction:
  `type TxGenerator era = [Fund] -> [TxOut CtxTx era] -> Either String (Tx era, TxId)`
* Put the new funds in a wallet:
  `type FundToStore = [Fund] -> IO ()`

The five arguments of `sourceToStoreTransaction` are
the callback functions for the five steps in the list above.
`FundSource` removes Funds from the wallet
and `FundToStore` adds new Funds, and those two functions are `IO` actions.
When sending transaction to multiple nodes,
each node-to-node submission protocol runs in a separate thread
and each thread calls `sourceToStoreTransaction` independently.

## Streaming abstractions

## Gotchas
There are many !

## JSON Config Language

## Nix and the tx-generator-service definition


## Fund Splitting and Benchmarking patterns

Patterns for benchmarking scripts
### Setup
### Tree based splitting
### Transaction targets
### Era transitions
### Concurrent/ Mixed workloads.

## NodeToNode Submission
