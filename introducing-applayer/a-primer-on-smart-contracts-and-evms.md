---
description: Detailing the building blocks of DeFi and the hydra we're trying to slay
---

# A Primer on Smart Contracts and EVMs

Blockchains like [Ethereum](https://ethereum.org), [Avalanche](https://support.avax.network/en/articles/5417030-what-is-the-ethereum-virtual-machine-evm), [Tron](https://developers.tron.network/v4.4.2/docs/tvm) and [Bitcoin Cash](https://cashscript.org/) support an ecosystem of *smart contracts* - pieces of code that run on the network and enforce certain rules or specific logic for given operations. They act much like real-world contracts, but are inherently programmable, allowing them to do much more than traditional paper contracts.

Smart contracts act as a foundation to build decentralized applications and concepts such as DeFi and blockchain gaming, while eliminating the necessity of "trust" usually provided by a third-party on a traditional real-world contract. You can read more about the topic in [Ethereum's website](https://ethereum.org/en/smart-contracts/). The most widely known implementation of such ecosystem is Ethereum's Virtual Machine ([EVM](https://ethereum.org/en/developers/docs/evm) for short), which pioneered and popularized the concept.

## Limitations of the EVM and current solutions

One of the biggest pain points of blockchain development since its inception is related to *performance*. Blockchains that use virtual machines share the same conceptual problem: contracts built on top of them often have limited speed and flexibility. This happens not only due to the contracts being coded in higher-level languages (as opposed to lower-level languages which are by nature more performant, although harder to handle), but also due to the nature of the virtual machines themselves, as they're built to be "generic computers with limited throughput".

This is common on Ethereum and other EVM-based chains - most major scaling solutions use the same unsustainable technology, which introduces the same issues of scalability in every deployed layer. Having to share a "generic computer" with the whole world is *inefficient by design*. Forcing a chain to be both generic and decentralized at the same time puts heavy limits on which types of applications you can decentralize and how much you can decentralize them.

As an example, in the case of the Ethereum Virtual Machine, you can *not* do any of the following:

* Loop a function more than 50 times due to block gas limit constraints;
* Have a stack size larger than 16 variables due to constraints on the EVM itself;
* Parallelize multiple contract calls (every time a new block has multiple transactions that interact with multiple different contracts, you have to load the contract, parse and save changes to the database for *each* single one of these contracts, *in order*).

As quoted by [Itamar](https://github.com/itamarcps): *"The biggest problem is that everyone is sharing the same computer, and that computer is a Commodore 64"*.

## Seeking a new solution

If the problem is inherently tied to the EVM, then... *why don't we just get rid of it?* This seems like a reasonable solution, but it can't be the only one.

Given the current blockchain landscape is primarily dominated by the Solidity EVM, a widely established standard that has spanned over a decade, it is unrealistic for many Web3 companies and developers to completely remove it from existing applications. However, to properly scale a Web3 based application, we must venture outside the virtual machine.

Thus, we'te taking an approach that is a hybrid of old and new: a modular blockchain that not only allows for *natively-coded contracts*, but also a natively-coded, performance-centric EVM with [*stateful pre-compiles*](https://medium.com/@AppLayerLabs/stateful-precompiles-evm-game-changers-or-another-overhyped-complexity-b064145b290e). In other words, a blockchain that doesn't *need* a virtual machine to run smart contracts, but still has support for existing contracts to be deployed as-is. Running an EVM with stateful pre-compiles unlocks performance optimizations to existing Solidity contracts and accelerates them with pre-compiled functions within the state, programmed in common performance-driven development languages such as C++, C#, Rust, and more.

This approach brings a tremendous advantage, however, it also comes with its own set of problems:

* It can be tricky to push new smart contract code into the network as a pre-compile. If you want to add or remove logic from your contract in the network, you have to force a mandatory update to all node operators - this could take *days*, which is a pretty big deal depending on how mission-critical the update is
* An application-specific pre-compiled contract might be too expensive to be natively executed on 1st party validation
* If the pre-compile is not hard-coded into the network and validated from a 3rd party, it must be ensured that this communication remains secure