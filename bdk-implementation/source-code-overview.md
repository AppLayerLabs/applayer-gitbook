---
description: An aerial view of the BDK's source code
---

# Source code overview

Looking at a higher level of abstraction, the original C++ implementation of the BDK has its code structured as several modules contained in the following folders:

* `src/bins`: code for the project's main binaries in their respective subfolders - the blockchain executable itself, contract ABI generator, network simulator, faucet API, test suite, etc.
* `src/bytes`: code related to the `bytes` namespace - a collection of structures, classes and helper functions for manipulating raw byte data
* `src/contract`: everything related to smart contracts - from ABI parsing to custom variable types and template contracts
* `src/core`: the heart of the BDK, contains the main components of the blockchain
* `src/libs`: third-party libraries not inherently tied to the project but used throughout development
* `src/net`: everything related to networking, communication between nodes and support for protocols such as gRPC, HTTP, P2P, JSON-RPC, etc.
* `src/utils`: commonly-used functions, structures, classes, overall logic and miscellanea to support the functioning of the BDK

There is also a `tests` folder that contains several unit tests for the components described above, following the same subfolder structure for simplicity purposes. You can use the `tree` command on a terminal in the project's root for more details on the folders' structures.

## Key components

As the project is constantly iterated upon and components may come and go with time, we rely mostly on self-documenting code (e.g. the Doxygen docs as stated earlier) to carry the burden of detailing them, so we recommend sticking with that if you need a deeper look on what each component does.

From a simpler point of view however, we can pinpoint the most important and/or most frequently used components across the entire project as follows:

### src/utils

* **Byte and Bytes** - aliases for the raw byte data types used throughout the project
* **ContractReflectionInterface** - namespace with utility functions that enable [reflections](https://en.wikipedia.org/wiki/Reflective\_programming) in C++ by extensive use of templates, used mainly for registering contract classes
* **DB** - abstraction of a [Speedb](https://github.com/speedb-io/speedb) database used internally for manipulating various kinds of storage data (including helper components)
* **FinalizedBlock** - abstraction of a block's structure and data in the chain, inherently unmodifiable after creation (ensures state integrity across nodes)
* **FixedBytes and derivatives** (Address, Functor, Hash, Signature, StorageKey) - abstraction of a C++ STL-compliant fixed-size byte array (e.g. `FixedBytes<10>` has *exactly* 10 bytes - derivatives have predefined sizes)
* **Hex** - abstraction of a strictly hex-formatted string (`0x[1-9][a-f][A-F]`)
* **JsonAbi** - namespace responsible for managing and converting contract ABI data to JSON format (used by the contract ABI generator binary)
* **Merkle** - custom implementation of a Merkle Tree, adapted from [those](https://medium.com/coinmonks/implementing-merkle-tree-and-patricia-tree-b8badd6d9591) [sites](https://lab.miguelmota.com/merkletreejs/example/)
* **Options** - singleton with data about the node, frequently accessed by the BDK (generated during build time from a .in file)
* **RandomGen** - implementation of the deterministic randomness generator used for almost everything related to blockchain consensus
* **SafeHash and FNVHash** - custom implementations for unordered map key hashing and [Fowler-Noll-Vo](https://en.wikipedia.org/wiki/Fowler%E2%80%93Noll%E2%80%93Vo\_hash\_function) hashing respectively across the entire project
* **Secp256k1** - namespace that abstracts the functionalities of Bitcoin's [secp256k1](https://en.bitcoin.it/wiki/Secp256k1) elliptic curve cryptography library, used for signing, verification and key derivation (including helper aliases)
* **TxBlock and TxValidator** - abstractions for a block transaction and a Validator transaction respectively (derived from Ethereum's "Account" model, contrary to Bitcoin's "UTXO" model) - includes a **TxAdditionalData** struct with metadata about a deployed contract (e.g. tx hash, gas used, call status, contract address, etc.)
* **UintConv, IntConv, StrConv and EVMCConv** - namespaces related to aliases, conversion and manipulation of specific types of data (e.g. integers, strings, EVMC data, etc.)
* **Utils** - namespace for miscellaneous utility functions, namespaces, enums and typedefs used across the project

### src/contract

See the Contracts section and the respective subsections for more details on the following components:

* Subfolders for the implemented precompiled contract templates and SafeVariables (see Precompiled contracts)
* **ABI** - namespace for handling Solidity ABI types and data encoding/decoding (see Solidity ABI)
* **BaseContract and DynamicContract** - base types for Protocol and Dynamic Contracts respectively (includes the **ContractGlobals and ContractLocals** helper classes for managing contract metadata)
* **ContractHost** - class responsible for managing contract execution stacks from beginning to end (including nested calls), the EVMC host implementation for EVM contracts, and general native<->EVM contract interoperability
* **ContractManager** - class responsible for managing all Dynamic Contract creation and deployment logic in the chain (includes the **ContractFactory** helper class)
* **ContractStack** - class responsible for managing temporary data from contract nested call chains (e.g. altered contract balances and SafeVariable commit/revert logic), used by ContractHost in a 1:1 ratio (only one stack instance spawned per host instance during a call)
* **ContractTypes** - list of custom contract names to instance during blockchain deploy (see Creating a Dynamic Contract > Deploying and testing)
* **Event** - abstraction of a Solidity event's structure and data

### src/core

* **Blockchain** - the entry point of the system and the main class that unites all the other components (basically the "power button on AppLayer's PC case")
* **Syncer** - class responsible for syncing the chain with other nodes in the network, handling transaction broadcasts and block creation (if the node happens to be a Validator)
* **Consensus** - class responsible applying the network's consensus rules during block and transaction processing
* **Dumpable, DumpManager and DumpWorker** - classes that respectively abstract a dumpable object (that is able to dump itself from memory to disk, e.g. blocks, transactions, contracts, the state itself, etc.), a list of said objects to be dumped when required, and the separate worker thread that does the actual work of dumping the objects
* **State** - abstraction of the chain's current state of accounts, balances, nonces, transactions, token balances, deployed contracts and other kinds of shared data at the current block in the network
* **Storage** - abstraction of the chain's block storage history, including transactions, contracts, accounts, emitted contract events and other immutable blockchain-related data (managed both in memory and disk)

### src/net/p2p

* **Broadcaster** - specifically responsible for handling broadcast messages (e.g. a new block being broadcast to all nodes)
* **ManagerBase and derivatives** (ManagerNormal, ManagerDiscovery) - responsible for managing a list of several session connections, their inter-communications and lifecycles
* **NodeConns** - abstraction of a list of currently connected and regularly refreshed peer nodes and their metadata
* **Session** - class that represents a TCP connection with another node, effectively communicating and sharing data with it
