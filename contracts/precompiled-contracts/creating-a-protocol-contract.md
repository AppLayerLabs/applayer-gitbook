---
description: A primer on what to expect when creating Protocol Contracts
---

# Creating a Protocol Contract

Protocol Contracts do not offer the same level of ease of use and security as Dynamic Contracts. However, by sacrificing some of these features, they enable more complex functionalities that would be impossible to achieve with Dynamic Contracts alone. For instance, Protocol Contracts can call functions without an active transaction call when processing a block, access system files and make requests to other nodes.

Introducing this new layer for processing information inherently carries risks, particularly in a decentralized network. To maintain a stable network, you must ensure that every node can execute the same operation, given a previous context. In a typical VM blockchain, this is achieved by allowing operations to be callable only by transactions and packaging these transactions into a block. When processing a block, all transactions that call a contract attempt to execute, ensuring that all operations are performed consistently within the given context.

As a developer of a Protocol Contract, it is crucial to manage the processing of information in a way that ensures different nodes do not arrive at different results when given the same context. The challenge lies in defining "context". In a VM blockchain, the context can be defined based on a block and all its past blocks. However, in AppLayer, Protocol Contracts can process beyond a transaction call and are actively changing their own context. Consequently, it is essential to guarantee that the context remains consistent across all nodes.

By adhering to the following best practices, you can create Protocol Contracts that maintain consistent behavior across nodes, and ensure the stability and security of your blockchain network. Remember that the key to developing robust Protocol Contracts lies in managing the context and ensuring that all nodes in the network can process information the same way.

We highly recommend using [Doxygen](https://doxygen.nl) to generate the documentation of the current source code, so you can better understand the structure of the project and how your contract can fit in.

## Clearly define the context

Establish a clear definition of the context for your Protocol Contract and ensure that this definition is understood and adhered to by all nodes in the network.

## Use deterministic processes

Ensure that any processes or operations within the Protocol Contract are deterministic, meaning they will always produce the same output when given the same input. This will help prevent discrepancies between nodes.

## Limit external dependencies

Minimize the reliance on external data sources or systems that could introduce variability into the context, making it more challenging to maintain consistency across nodes.

## Implement thorough testing

Test your Protocol Contract extensively under various conditions and scenarios to identify potential issues and vulnerabilities. This includes simulating different network conditions and node configurations to ensure consistent behavior.

## Maintain clear documentation

Document your Protocol Contract design, functionality, and context requirements in a clear and concise manner. This will help other developers understand how to interact with the contract and maintain consistency across nodes.

## Monitor and update

Continuously monitor the performance and security of your Protocol Contract after deployment. Address any issues or vulnerabilities that may arise and implement updates as necessary.

## Add the Protocol Contract to the protocolContractAddresses map

Under `src/contract/contractmanager.h`, you will find a global map which contains the registered Protocol Contract addresses. You should stick to the same format as the other contracts, and add your contract address to the map.

## Pay attention to the contract's state

It is required to pay attention to the state of the contract and how nodes will interact between each other. For example, if you have a contract that is doing parallel processing of a task, you need to make sure that this task can be replicated with the same result in any node. Not taking care of the state of a contract will eventually lead to undefined behavior, and that's a hornet's nest nobody wants to touch.

## Override `ethCall()` and take care of your revert conditions

It is necessary to override the `ethCall()` function in order to make functions callable by a transaction or an RPC `eth_call`. Protocol Contracts don't have automatic function parsing, neither handle call reverts, so you need to handle it all yourself.

## Pay attention to the contract state's commit status

When the contract state's `commit` flag is set to `false`, it means that the current call is trying to simulate if it's going to throw or not, but if `true`, it means that the current call is trying to commit to the state, and you should do it respectively if it doesn't throw.

## Add a reference to `ContractManager`

It is necessary to add a reference of your Protocol Contract to the `ContractManager`. This is done within the `rdPoS` contract (another Protocol Contract). See more under these files:

* `src/contract/contractmanager.h`
* `src/contract/contractmanager.cpp`
* `src/core/rdpos.h`
* `src/core/rdpos.cpp`
