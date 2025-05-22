---
description: A primer on how P2P messaging works in the BDK
---

# P2P Overview

This subchapter provides a comprehensive overview of the life-cycle of a P2P connection within the BDK and the dynamic flow of data between nodes.

## Keeping track of peers

Decentralized networks work with the concept of *peers* - computers in a network running the same kind of software that connect to each other without any kind of third-party in between (e.g. a torrent client that seeds its files to other computers running the torrent protocol). This is one of the essential pillars of the crypto ecosystem - the more active peers a network has, the stronger and more resilient it is against malicious attacks, power failures and other similar issues.

BDK-powered chains make use of the **NodeConns** class to maintain and periodically update a list of all peer nodes in the network that are connected to the local node, as well as their respective metadata. It is through this list that the local node keeps itself up-to-date with the most recent node info possible, while also dropping stale connections and timed out peers every now and then to avoid potential network sync problems.

## Communicating with other nodes

The **Session** class encapsulates a TCP connection with a remote node, being responsible for managing handshakes, sending and receiving messages by reading and writing data through sockets. It serves for both client and server connections, having queues for both inbound and outbound messages, allowing any thread that is responsible for sending a message to carry on with its task without having to wait for the message transmission to complete.

Once a session has successfully established a connection with a remote node, it is added to a list of sessions controlled by an internal manager (see below). It is critical to properly manage those active sessions, as destroying the session will also destroy its socket - this design choice is on purpose, since if communication still occurs through a "dead" socket from a session that no longer exists, this will lead to a program crash.

## Life cycle of a session

Upon instantiation, a session's life-cycle is composed of three different routines:

* **Handshake**: first of all, the session gives a handshake to the remote endpoint, effectively making a connection so data can be shared between both nodes
* **Read**: after the handshake, the session listens indefinitely for inbound messages from the remote endpoint. When a message arrives, the session reads it and, if required, proceeds to write a response to the remote endpoint
* **Write**: if the remote endpoint sent a message that requires a response, the session does exactly that - it writes an outbound message for the endpoint and sends it, then goes back to listening indefinitely for the next inbound message

While the handshake routine is done only once, the read and write routines are executed cyclically through the entire duration of the connection, until it is properly closed by one of the endpoints.

## Managing sessions

Sessions are managed by the **ManagerBase** class, which acts as the backbone of the BDK's P2P networking stack. It bears the responsibility of managing sessions, their respective sockets and the global I/O context used by them, overseeing their operations and serving as a base for specialized classes according to th node's type (see below).

Once a session has successfully completed a handshake, it is registered within the manager, which then keeps track of that session's lifecycle. The manager's responsibilities include maintaining a registry of active sessions, handling incoming and outgoing requests and responses, and maintaining the communications between them.

Given its extensive duties, it's imperative that the functions within the manager remain as "active" or "lightweight" as possible - as in, ensure their mutexes are not locked for extended periods, as the manager is concurrently accessed by multiple threads to (de)register sessions, parse messages and/or request information from other nodes. If the manager stays locked for a long time, the node risks being blocked altogether, with potential repercussions extending to the entire network.

It's also important to be aware of the lifespan of the manager's I/O context - sessions do not manage their own isolated context, instead they all use the manager's. This is done for performance purposes, but a greater care must be taken, since if it is deleted at some point and then operations are performed on it (through means of a pointer which would be stale at that point), an exception could trigger when using the now-dangling pointer because the actual I/O context object it referred to would've been already destroyed.

## Node types

As said before, the **ManagerBase** class is not used by itself, rather it is derived into two other specialized classes: **ManagerNormal** and **ManagerDiscovery**. They are aptly named after the two types of nodes that compose a BDK-powered network: *Normal* and *Discovery* nodes respectively.

Normal nodes act as regular nodes in a blockchain, receiving and processing blocks and transactions, while Discovery nodes have the sole purpose of transmitting info about other nodes connected to it to the rest of the network. Due to our usage of CometBFT (which already has its own P2P stack), we have disabled Discovery nodes entirely for now, with plans of reimplementing them in the future.

## Message types

Every incoming message from a remote endpoint is promptly parsed by the manager and can fall into one of the following categories, with each one of them being treated distinctly (the names here are conceptual, the actual names in code differ a little bit):

* **Request** - a query for specific data from another node - e.g. a list of blocks/transactions, info about the node itself, etc.
* **Answer** - an answer to a given request
* **Broadcast** - a dissemination of specific data to the network *with* possible re-broadcasting ("flooding"), such as a new block or transaction
* **Notification** - a dissemination of specific data to the network *without* re-broadcasting, such as node metadata

*Request* and *Answer* messages work together in a bidirectional flow that goes like this:

* The sender node initiates a Request by generating a random unique 8-byte ID for reference, registering it internally and sending it alongside the message
* The receiver node receives the Request, its manager parses it and formulates an Answer with the requested data, assigns it the same 8-byte ID from the Request and sends it back to the sender node
* The sender node receives the Answer and checks if the received ID is the same one that was registered earlier. If it is, the manager matches the associated Request with the received Answer and deregisters the ID. If the ID is *not* registered, the Answer is discarded altogether

*Broadcast* and *Notification* messages work on their own in a simpler unidirectional flow instead, as the receiver node doesn't have to answer back to the sender. Instead, it verifies the received data and adds it to its own blockchain. The difference between both types is that *Notification* messages are never re-broadcast, while *Broadcast* messages may or may not be re-broadcast to other nodes, depending on whether said nodes had already received or not said broadcast in the past.

Due to this specific condition, Broadcast messages are specifically handled by the **Broadcaster** class, while the other types are handled normally by ManagerBase and its derivative classes.

## Asynchronous Message Parsing

To optimize the performance of the manager's I/O context and avoid any kind of bottleneck, message parsing is offloaded to a separate thread pool. The first thread under our control that accesses the message is the I/O context itself executing that particular session. The thread pool then handles both parsing of the message and writing back to the session, which involves adding tasks to its write strand or queue. The function that handles the answer to a message always returns a promise with the answer. A thread that called a request within the manager towards another node will wait for a few seconds or until the answer is received.

One performance-enhancing strategy we employ is the use of pointers for handling each message. This prevents unnecessary copying and provides significant benefits when dealing with broadcasts. In such cases, a single message can be utilized by multiple writing sessions, thereby offering a performance boost to the network.

Our current C++ design uses `shared_ptr` for all messages, however, we plan to transition to a system where `unique_ptr` is used for inbound messages and `shared_ptr` for outbound messages. The rationale behind this is to better handle memory ownership. The use of `unique_ptr` provides clearer ownership semantics and improved performance. Since `unique_ptr` cannot be moved into a function (which would be the task posted to the thread pool), we are simply using `shared_ptr` for now.
