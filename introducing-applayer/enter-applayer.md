---
description: The light at the end of the tunnel
---

# Enter AppLayer

What if there was a blockchain that incorporated stateful pre-compiles, allowing third parties to deploy and natively support these contracts within a single network that shares its state across all nodes?

This is exactly what AppLayer does. It is a modular blockchain with multiple layers, including an EVM, stateful pre-compiles, and chain abstraction.

The AppLayer Network is made up of three parts:

* A Blockchain Development Kit (hereby denominated [**BDK**](https://github.com/AppLayerLabs/bdk-cpp)), with extensive documentation for developers to easily build their own app-specific chains with unprecedented freedom
* An EVM network built on top of the BDK which enables builders to deploy EVM smart contracts and scale with C++ stateful pre-compiles
* A network that allows bridging data and assets between app-specific chains and external chains called the Chain Abstraction Network (**CAN**)

Therefore, blockchains built using the BDK are able to communicate with each other through AppLayer.

## Key Features

AppLayer brings a myriad of options for blockchain developers such as:

* **Natively-coded contracts** - developers can code their contracts directly in the blockchain's native language, bypassing extra layers of abstraction to leverage the full potential of AppLayer
* **Out-of-the-box support for Solidity contracts** - those who already have contracts coded in Solidity can use AppLayer's built-in, natively-coded EVM as a drop-in replacement
* **Stateful pre-compiles** - Solidity contracts can use stateful pre-compiles from our built-in EVM to achieve a higher performance compared to conventional EVMs
* **State-of-the-art consensus protocol** - our consensus engine is currently powered by [CometBFT](https://cometbft.com/), but we have plans for expanding beyond it (see the Future Plans section for more info)
* **Validators** - our network is executed and secured by a type of node we call "Validator", responsible for creating, gathering and signing data on blocks, as well as generating a "randomness" seed used to select the next block creator in the chain

## Potential use cases

AppLayer provides all the essential tools to build a wide range of products, powering applications such as:

* **DeFi**: build financial products with security and ease. Leveraging AppLayer's performance network enables a whole new generation of DeFi products such as facilitating millions of trades per second
* **Data Storage**: AppLayer makes data storage easier, more affordable, and secure. Builders can backup whole ledgers or fully decentralize any form of a database natively in the AppLayer network
* **GameFi**: gaming projects can use a performance-driven infrastructure to build game engines capable of leveraging a pure blockchain solution with a whole new range of in-game features

Other potential use cases include (but not limited to):

* Multiplayer games/servers
* Decentralized exchange
* Decentralized and hyper-available caching for dApp databases
* Decentralized e-mail
* HR portal
* VPN
* Cloud services
* Video rendering
* E-commerce
* Arbitrage bot
* Blockchain-enabled utilities such as water and power
* Supply chain and logistics
