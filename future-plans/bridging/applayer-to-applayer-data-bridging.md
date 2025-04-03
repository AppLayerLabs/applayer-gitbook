---
description: How native chains would exchange data on AppLayer
---

# AppLayer-to-AppLayer Data Bridging

Here's how bridging arbitrary data from an AppLayer chain (A) to another (B) would work:

* Chain A sends a request in the upcoming block and communicates this, along with the block reference, to the Chain Abstraction Network. This request is written to the next block and relayed to AppLayer's Chain Abstraction Network.

<figure><img src="../../.gitbook/assets/Diagram 1.0.png" alt=""><figcaption><p>A sends a request to the Chain Abstraction Network</p></figcaption></figure>

* A set of Validators and Sentinels are randomly selected using RandomGen (which works exactly the same way in all nodes in the network). The selected Validators and Sentinels review Chain A's request and forward it to Chain B.

<figure><img src="../../.gitbook/assets/Diagram 2.png" alt=""><figcaption><p>Validators and Sentinels in the Chain Abstraction Network relay A's request to B</p></figcaption></figure>

* Chain B receives the request from the Chain Abstraction Network and retrieves the data from its chain, encapsulating it within a merkled item. This data and its internal reference are subsequently transmitted back to the Chain Abstraction Network for permanent storage.

<figure><img src="../../.gitbook/assets/Diagram 3.png" alt=""><figcaption><p>B gathers data from itself and sends it back to the Chain Abstraction Network</p></figcaption></figure>

* Validators and Sentinels check and corroborate the data submitted by Chain B with other nodes from Chain B to validate its presence in the designated block.

<figure><img src="../../.gitbook/assets/Diagram 4.png" alt=""><figcaption><p>Chain Abstraction Network checks other B nodes to verify data</p></figcaption></figure>

* Once the data is verified, Validators and Sentinels sign the data and publish it inside the Chain Abstraction Network, while also relaying it back to Chain A along with their signatures.

<figure><img src="../../.gitbook/assets/Diagram 5.png" alt=""><figcaption><p>Chain Abstraction Network relays signed data back to A</p></figcaption></figure>

* Chain A verifies the signatures and checks if the randomly selected nodes were using the network's RandomGen seed. If everything matches, the exchange is complete, but if not, there's a malicious node in the network which should be reported.

It is possible for Chain A to only send information to the target chain without having to wait for an answer. This call can possibly trigger logic within the target chain, depending on how its developers decided to handle the message.

