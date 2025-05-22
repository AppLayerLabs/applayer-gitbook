---
description: How are precompiled contracts managed within the BDK
---

# Managing precompiled contracts

Contracts in the BDK are managed by a few classes working together (check the `src/core` and `src/contract` folders for more info on each class):

* `State`  (`src/core/state.h`) is responsible for owning all the Dynamic Contracts registered in the blockchain (which you can get a list of by calling the class' `getCppContracts()` and/or `getEVMContracts()` functions, depending on which ones you need), as well as their global variables (name, address, owner, balances, etc.)
* `Storage` (`src/core/storage.h`) is responsible for properly storing contract data, such as emitted events and the transactions that triggered them
* `ContractManager` (`src/contract/contractmanager.h`) is responsible for solely creating, registering and passing contracts to the state (with the `ContractFactory` namespace providing helper functions to do so)
* `ContractHost` (`src/contract/contracthost.h`) is responsible for allowing contracts to interact with each other and enabling them to modify balances - this kind of inter-communication is done by intercepting calls of functions from registered contracts (if the functor/signature matches), which is done through either an `eth_call` request or a transaction processed from a block
* `ContractStack` (`src/contract/contractstack.h`) is responsible for managing alterations of data, like contract variables and account balances during nested contract call chains, by keeping a stack of changes made during a contract call, and automatically committing or reverting them in the account state when required (for non-view functions)

The `ContractManager` class is a *Protocol Contract* by itself, but it does not own or create any Protocol Contracts - they are created during blockchain initialization, and a reference to each of them is stored within the class, allowing it to access them directly. Dynamic Contracts, however, are fully owned and stored by the state in an internal map. This ensures that each contract has a unique address, which is derived using a similar scheme as an EVM.

Because of this, we don't necessarily need to know the type of the contract stored within the pointer, only during either creating it or loading it from the database. This is why all contracts inherit the `BaseContract` class, as it contains a `name_` variable specifying the name of the contract. This name must be the same as the name of the class, as it is used to identify the contract during loading and creation.

Using `ContractHost` for contract inter-communication instead of accessing the state directly allows us to do it in an isolated and secure way. For example, if we want to send tokens from a contract to another address, we don't access the balance directly from the state. Every time a function is called by a **user** (not another contract), the balances map available for contracts is empty, only populating what is currently being accessed and not previously available. When modifying the balance, only the mapping within `ContractStack` is modified, therefore allowing for multiple nested contract functions to revert in an atomic fashion while not affecting either the state or other contracts altogether.

## The BaseContract class

The `BaseContract` class, declared in `src/contract/contract.h`, is the base class from which all contracts derive from. This class holds all the [Solidity global variables](https://docs.soliditylang.org/en/v0.8.17/units-and-global-variables.html), besides variables common among these contracts (such as contract address). Have a look at the header file for further reference on its structure.

Regarding the `callContractWithTransaction` and the `ethCallContract` functions in the previous example, the former is used by the state when calling from `processNewBlock()`, while the latter is used by RPC to answer for `eth_call`. Strings returned by `ethCallContract` are hex strings encoded with the desired function result.
