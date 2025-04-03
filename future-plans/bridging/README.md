---
description: How bridging would work on AppLayer
---

# Bridging

There are inherent flexibility issues with **native** and **application-specific** chains when compared to traditional EVM chains. An application-specific chain is inherently limited as it can not support a complete service that involves interaction between more than one application. Those problems could heavily damage the reputation of a project built on them.

Our conceptual solution to this issue is to allow AppLayer-enabled blockchains to natively communicate with each other by using the Chain Abstraction Network (**CAN** for short) as a middleman, where AppLayer serves as an intermediary between two dApp chains trying to communicate with each other. We call that **bridging**.

At the moment, bridging in AppLayer is implemented in a centralized manner, much like other conventional networks. Eventually it would evolve to a fully decentralized solution, which would allow bridging for *arbitrary data* and *tokens*, both *between AppLayer nodes* and *between AppLayer and external networks*.

## How would safety be ensured?

The Validator (and eventual Sentinel) nodes that read from a given chain are determined using *RandomGen* - a trustless, decentralized randomness generator developed by AppLayer Labs. We would ensure and keep a fair selection of nodes, however, there could be a possibility of a 51% attack.

For example, in a network with 100 nodes, if a malicious user controls 51 of them, and all of them get selected for driving a cross-chain request and a block, they could collude and forward any message they want. We would avoid this by introducing Sentinels to the network, which would ensure this collusion does not happen by working together with Validators to fortify the network’s security.


