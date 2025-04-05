---
description: Now it's time to take it for a ride!
---

# Connecting the wallet

With your local testnet node running, it is now possible to configure and connect your preferred Web3 wallet to it and play around with an AppLayer-powered blockchain. As said at the beginning, we recommend using [Metamask](https://metamask.io) as it is the most popular one, but you're free to use any other client you wish.

As an example, here's how to configure MetaMask to connect to your local testnet:

| Field           | Value                                           |
| --------------- | ----------------------------------------------- |
| Network Name    | AppLayer Local Testnet                          |
| New RPC URL     | [http://127.0.0.1:8090](http://127.0.0.1:8090/) |
| Chain ID        | 808080                                          |
| Currency Symbol | APPL                                            |

<figure><img src="../.gitbook/assets/metamask (1).png" alt=""><figcaption></figcaption></figure>

Once you're connected, import the following private key for the chain owner account:

```
0xe89ef6409c467285bcae9f80ab1cfeb3487cfe61ab28fb7d36443e1daa0c2867
```

This account contains a huge number of APPL tokens from the get go and is able to call the `ContractManager` contract, deployed at the address:

```
0x0001cb47ea6d8b55fe44fdd6b1bdb579efb43e61
```

Other details about the deployed testnet chain can be found in the project's README.md file.