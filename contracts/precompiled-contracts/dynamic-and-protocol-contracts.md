---
description: Which kinds of precompiled contracts can be coded and managed with the BDK
---

# Dynamic and Protocol Contracts

AppLayer's BDK offers two main classes of contracts: **Dynamic Contracts** and **Protocol Contracts**. The differences between both classes come from how they are created and managed within the BDK.

## Dynamic Contracts (recommended)

Dynamic Contracts are the recommended type for usage with AppLayer. They are *managed by the state* and can only be created by `ContractManager`, which enables the chain owner to create an unlimited number of contracts.

They can also *use special types called **SafeVariables*** (explained further) - an additional layer of protection that allows better control over whether variable changes are committed to the state or automatically reverted when necessary (e.g. when a transaction fails).

Dynamic Contracts can *only be called when a block is being processed* and are *directly loaded into memory*, working very similarly to Solidity contracts (including *event emission*).

## Protocol Contracts

Protocol Contracts are *directly integrated into the blockchain*, therefore not linked to the `ContractManager` class and not contained by the state, which removes some restrictions but adds others (explained further).

This makes them able to be designed to *process information beyond transaction calls*, like communicating directly with other nodes, accessing files within the current system, and even automatically calling themselves when certain conditions are met - anything is possible, as long as you don't break your own blockchain (more on that later).

As a downside to that, unlike Dynamic Contracts, they *cannot use SafeVariables nor emit events*. This means it's up to the developer (for the most part) to handle the contract's variables and their commit/revert logic within the blockchain's source code.
