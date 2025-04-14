---
description: Which kinds of precompiled contracts can be coded and managed with the BDK
---

# Dynamic and Protocol Contracts

AppLayer's BDK offers two main classes of contracts: **Dynamic Contracts** and **Protocol Contracts**. The differences between both classes come from how they are created and managed within the BDK.

## Dynamic Contracts (recommended)

Dynamic Contracts are the recommended type for usage with AppLayer. They are *managed by the state* and can only be created by `ContractManager`, which enables the chain owner to create an unlimited number of contracts.

They can also *use special types called **SafeVariables*** (explained further) - an additional layer of protection that allows better control over whether variable changes are committed to the state or automatically reverted when necessary (e.g. when a transaction fails).

Dynamic Contracts can *only be called when a block is being processed* and are *directly loaded into memory*, working very similarly to Solidity contracts (including *event emission*).

AppLayer's BDK provides ready-to-use templates for the following Dynamic Contracts:

* `ERC20` (template for an ERC20 token)
* `ERC20Wrapper` (template for an ERC20 wrapper)
* `ERC721` (template for an ERC721 token)
* `NativeWrapper` (template for a native asset wrapper)
* `DEXV2Factory` (template for a DEX factory)
* `DEXV2Library` (namespace for commonly used DEX functions)
* `DEXV2Pair` (template for a DEX contract pair)
* `DEXV2Router02` (template for a DEX contract router)
* `UQ112x112` (namespace for dealing with fixed point fractions in DEX contracts)

Some contracts were converted directly from their OpenZeppelin counterparts and are also available for use:

* `ERC721URIStorage` (template for managing ERC721 token storage)
* `Ownable` (template for managing authorized access to certain contract calls)

There are also specific contracts that only exist for internal testing purposes and are not meant to be used as templates:

* `SimpleContract` (what it says on the tin - a simple contract, used for both testing and teaching purposes)
* `RandomnessTest` (contract for testing random number generation)
* `ERC721Test` (derivative contract meant to test the capabilities of the ERC721 template)
* `TestThrowVars` (contract meant to test SafeVariable commit/revert functionality using exception throwing)
* `ThrowTestA/B/C` (contracts meant to test nested call revert functionality)
* `SnailTracer` / `SnailTracerOptimized` (C++ conversions of the [SnailTracer](https://github.com/karalabe/snailtracer) contract, used for benchmarking purposes)
* `Pebble` (contract meant to be used in the testnet as a little NFT mining game)

## Protocol Contracts

Protocol Contracts are *directly integrated into the blockchain*, therefore not linked to the `ContractManager` class and not contained by the state, which removes some restrictions but adds others (explained further).

This makes them able to be designed to *process information beyond transaction calls*, like communicating directly with other nodes, accessing files within the current system, and even automatically calling themselves when certain conditions are met - anything is possible, as long as you don't break your own blockchain (more on that later).

As a downside to that, unlike Dynamic Contracts, they *cannot use SafeVariables nor emit events*. This means it's up to the developer (for the most part) to handle the contract's variables and their commit/revert logic within the blockchain's source code.
