---
description: The solution to consensus on performance-driven blockchains
---

# rdPoS

To keep consensus on such a blazing fast network without tripping up and/or having to deal with rollbacks, it would be necessary to use a *random deterministic block creation* that allows only one given node to create a block for a given time, eliminating the risk of a block race condition in the network.

Going beyond the current consensus engine (CometBFT), AppLayer would ideally implement its own consensus algorithm, hereby denominated **rdPoS** (*Random Deterministic Proof of Stake*). It would empower Validators and Sentinels to deal with block congestion and random number generation. This section aims to explain in-depth how such algorithm would work and be used by the AppLayer protocol.

## A primer on blockchain rollbacks

One of the biggest problems of blockchain development is handling block rollbacks. For example, on the Bitcoin chain, assuming there is a latest block that has another block after it. If a node receives a block that replaces the latest block, the next block and all the transactions in it are replaced too, which results in a rollback of the blockchain’s state by one block.

The Bitcoin blockchain and other derivatives follow the "longest lived chain" rule (the chain with the most accumulated proof of work is the main chain). However, rollbacks unearth problems in that rule. For instance, when a developer is building dApps where they have to deal with such special conditions, it could take a greater effort depending on the size and/or complexity of the application.

<figure><img src="../.gitbook/assets/Diagram 6.png" alt=""><figcaption><p>In this example, block C is replaced by block D followed by block E, rolling back the transactions made in block C</p></figcaption></figure>

The ideal solution is to avoid the rollback condition altogether. This can be done by deterministically defining which network node can create a block, thereby eliminating the block race condition and keeping everyone in the network synchronized to the same latest block. This is where rdPoS comes in. It would pair a block congestion system and a random number generator system, allowing only one Validator to create a block at any given time, thus avoiding rollbacks and achieving consensus on ultra-fast networks.

## How rdPoS would work

The heart of rdPoS is *RandomGen*, a deterministic uint256\_t generator used for almost everything related to consensus. This deterministic randomness ensures that every node has a chance to respond to a given request (block, randomness, bridging, etc.), while making sure that the nodes selected from the network are truly random and not problematic nodes operated by a malicious actor.

For RandomGen to be viable, it needs to be seeded with a truly random number. This is how it works:

* Every time a new block is about to be created, 16 random nodes are selected using RandomGen with the previous block’s randomness seed
* These nodes make a 32-byte random string (`RandomnessSeed`) and hash it (`RandomnessHash`), then sign the hash and publish it to the network
* After all the nodes have signed and published their hashes to the network, they can publish the real data, verifying that no one is trying to manipulate the end result
* After the data is published and included in the block, the randomness seeds are concatenated and hashed, and the resulting hash is used to seed the next block creation

We must pay attention to the current state of RandomGen to ensure all nodes are always in the same internal state, so they can properly synchronize with each other.

## Creating a block under rdPoS

A block in an rdPoS network would be created by the following rules:

* A list of network Validators is randomly generated and sorted using the "randomness" seed from the previous block

<figure><img src="../.gitbook/assets/RandomListCreation.png" alt=""><figcaption><p>New random Validator list being created</p></figcaption></figure>

* The first Validator from the list will be the block creator, while at least 4 others will create a random 32-byte string and make two transactions with it: one containing the hash of said string, and another containing the string itself, both signed

<figure><img src="../.gitbook/assets/HashTransactionBroadcast.png" alt=""><figcaption><p>Validators performing a hash transaction broadcast</p></figcaption></figure>

<figure><img src="../.gitbook/assets/RandomTransactionBroadcast.png" alt=""><figcaption><p>Validators performing a random transaction broadcast</p></figcaption></figure>

* The hashes are verified to make sure they match their respective random strings

<figure><img src="../.gitbook/assets/HashKnowledgeProof.png" alt=""><figcaption><p>Transactions are checked against each other</p></figcaption></figure>

* A new block is created by the first Validator, concatenating and hashing the other Validators' random strings to create a new "randomness" seed that will be used at the next block

<figure><img src="../.gitbook/assets/BlockRandomness.png" alt=""><figcaption><p>New randomness seed is generated</p></figcaption></figure>

<figure><img src="../.gitbook/assets/NewBlock.png" alt=""><figcaption><p>New block is created with randomness seed, Validator signature and the transactions</p></figcaption></figure>

* The block is signed and published to the network by the first Validator, while the other Validators verify that all transaction signatures (random and hashed) correspond with the list created at the start
* The genesis block (the very first block in the chain) enforces a given fixed randomness to be valid, since there is no previous block before it to derive the randomness from. At least five hardcoded Validators are needed to bootstrap the network, as each block requires at least four Validators to confirm the string and hash transaction signatures, and one for signing the block itself

As quoted by [Supra](https://github.com/Jean-Lessa): *"It's like playing poker but everyone hashes their hands first before showing the real cards"*.

## Validator implementations

Under rdPoS, developers would get to choose how Validators are added to the network, based on three pre-established implementation options: *permissionless*, *permissioned*, and *semi-permissioned*.

### Permissionless

In a permissionless implementation, all Validators would have to participate in block creation to ensure that there’s no collusion. A totally permissionless network using rdPoS could face problems when the number of Validators in the network grows massively (e.g. around 10,000), because the latency between those nodes could increase significantly.

To solve this problem, the block time in a permissionless network should be bigger (e.g. 15-30 seconds) so all the nodes have enough time to respond. Validators could be added to a permissionless network by locking a certain amount of tokens in the contract that contains the rdPoS logic.

### Permissioned

In a permissioned implementation, every network would have a "master address" that could add as many Validators as desired. Whoever sets up this implementation would be responsible for keeping the chain up and running. The recommended number of nodes for this implementation would be at least 32, but more or less nodes could be used depending on the application’s needs.

### Semi-permissioned

In a semi-permissioned implementation, both Validators and Sentinels would be used in tandem. Validators would mirror the "permissionless" side (being added to the network by locking tokens), while Sentinels would mirror the "permissioned" side (being added to the network with a master address).

This implementation is different in the sense that neither Validators nor Sentinels can create a block on their own. The deterministic randomness would require at least one of the transactions from a Sentinel and whoever publishes the block would have to follow the Validator list order set during rdPoS processing.

This means the network could have a smaller number of Validators (e.g. 16 instead of 32), requiring less computing power for verification but remaining highly secure. As Sentinels take part in the process, every extra byte in the concatenated randomness seed will change the resulting hash, thus ensuring that any attempt of tampering is easily detected and dealt with.

## Slashing

What happens when a Validator answers with a "randomness" hash that does not match its own hash? Or when a Validator creates an invalid block with invalid transactions? Or when a Validator can't create a block before reaching the network's time limit?

Misbehaving nodes must suffer dire consequences. Since Validator signatures are required at protocol level, if a Validator tries to break the rules, it's possible to know who it is thanks to the signature, and "slash" it from the network.

The biggest problem with this is a group of Validators being "slashed" and halting network activity. This could be solved by adding extra conditions to the network - for example, if the network wants to change the current block creator (in case it's been "slashed"), at least 90% of the Validators in the network have to sign a transaction consenting with the change, always maintaining the majority's consensus.
