---
description: How Application Chains would work on AppLayer
---

# Application Chains

An **application chain**, also known as an AppLayer™, is a blockchain built using the BDK and deployed on the Chain Abstraction Network. An AppLayer™ would primarily enable developers to create a chain dedicated to a particular application, with the chain's rules specifically tailored to the security, performance, speed, decentralization and other service delivery needs of the application itself. These AppLayers would be responsible for securing their own ledger of accounts and balances along with their own execution logic.

AppLayer's BDK currently supports C++ and Solidity for blockchain development, with future plans for other languages such as Rust, C#, Golang, and more. These application chains would ideally compile to a binary to enable efficient execution in parallel to Solidity bytecode. Whoever sets up an application chain would be in charge of validating its own transactions and attracting others to run its Validators.

## Types of precompiled contracts

With this idea implemented, AppLayer would conceptually have three different types of contracts: *1st-party* , *3rd-party*, and *AppChains/AppLayers*.

**1st-party** contracts are contracts provided by AppLayer itself as ready-to-use templates in the `src/contract/templates` subfolder (e.g. ERC20, ERC721, etc. - what we have right now), officially supported and leveraging all the features of the blockchain. Those contracts are also available in the AppLayer EVM chain.

**3rd-party** contracts would act like a "slim Layer 2", in the sense that they would be only processed, but have no concepts of consensus, blocks, transactions, etc. unlike a normal contract. They would essentially act like a "daemon" of sorts, running atop the main chain (just like a Layer 2), reading transactions made on it and reacting accordingly if said transaction happens to call it. The transaction would be executed and the contract would publish the results back on the main chain (e.g. a user sent tokens to a 3rd-party exchange contract, once the transaction is confirmed on the main chain the contract reads the data field and executes its own logic, then sends the exchanged tokens back on the main chain as its own transaction).

**AppChains/AppLayers** themselves would act as a "full Layer 2" instead, having their own blocks, transactions, consensus, etc., but still depending on the main chain to execute their separate logic.