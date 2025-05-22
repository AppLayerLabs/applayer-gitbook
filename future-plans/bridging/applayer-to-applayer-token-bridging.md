---
description: How native chains would exchange tokens on AppLayer
---

# AppLayer-to-AppLayer Token Bridging

Token bridging between two AppLayer nodes would work the same way as arbitrary data bridging, but with extra checkups to make sure that a given chain could not mint another chain's tokens.

Due to how the network was designed, when doing a cross-chain transaction we can only ensure that the data *exists*, not that it is *valid in context*. That breach allows a given chain to mint the native token of another chain, because the Chain Abstraction Network does not verify if the token is valid inside that network.

We could avoid this problem by keeping a "token table", which is just a "spreadsheet" of chains and their external token balances. This doesn't include the given chain's own native token, since it can freely mint its own token itself and does its own internal validations to avoid invalid minting conditions.

For example, we have chains A, B, and C, each one with tokens of each other. The Chain Abstraction Network would keep track of:

* How many B and C tokens exist on A
* How many A and C tokens exist on B
* How many A and B tokens exist on C

When bridging another chain's tokens, the Chain Abstraction Network would check if that chain has enough balance to do so. When bridging your own tokens, the Chain Abstraction Network would only have to increase the balance at the target chain, since the `exit` transaction from your chain has to be included in one of your blocks, which means it has been verified and validated inside your own network, so there's no need to do it again from the outside.

This method of bridging would follow mint/burn mechanisms.

