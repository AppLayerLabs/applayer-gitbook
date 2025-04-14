---
description: Not all contract calls are equal
---

# Internal and external contract calls

In the context of interacting with smart contracts, calls can be classified into two types: *external* and *internal*.

**External calls** occur when a contract is invoked from an external source, which can be either a user transaction (when a block is being processed) or a remote procedure call (RPC for short, e.g. an `eth_call` request). External calls serve as entry points to the contract from the outside world, initiating a transaction or a query. Within AppLayer's BDK, to process an external call, a `ContractHost` object is created, which establishes the execution context based on the specified parameters.

**Internal calls** occur when a contract, already running as part of an ongoing transaction or chain of execution, calls another contract or a function within another contract. This is considered an internal process, as it happens within the chain of execution started by an external call. In AppLayer's BDK, internal calls use the same `ContractHost` object created for the initial external call. This maintains consistency and control over the execution environment, allowing for commits or rollbacks within the same transactional context.

Basically, an external call creates a new "chain of execution", while internal calls only exist within that same chain of execution, thus they cannot create new chains by themselves. A chain of execution is a sequence of calls that are executed in a given external call, always composed of an initial external call and a set of internal calls (if applicable).
