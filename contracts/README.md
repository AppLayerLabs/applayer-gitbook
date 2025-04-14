---
description: How smart contracts work in AppLayer
---

# Contracts

Smart contracts, in short, are custom developer-made classes that directly interact with the current state of the blockchain. This chapter will comprehensively cover the details of smart contract usage and development in the AppLayer network - going from creating precompiled, natively-coded contracts using the BDK, to operating the AppLayer EVM to leverage existing Solidity contracts.

AppLayer ensures that contracts deployed in its network, no matter the type, remain compatible with existing frontend Web3 tools (e.g. [MetaMask](https://metamask.io/), [ethers.js](https://github.com/ethers-io/ethers.js/), [web3.js](https://docs.web3js.org/), etc.). Those are originally designed to interact with Solidity contracts and thus require a similar interface.

To call your contract's functions from a frontend, you'll also need to generate its ABI - you can either do it directly with our generator tool if coding a pre-compiled contract (explained further), or replicate their definitions in Solidity and use an external tool like Ethereum's [Remix](https://remix.ethereum.org/) or any other of your preference. This ABI can then be used by your preferred Web3 frontend.