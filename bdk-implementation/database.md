---
description: How the BDK's internal database is structured and how data is stored in it
---

# Database

BDK nodes use an in-disk database for storing data about themselves and other nodes in the network, such as blocks, transactions, contracts, node metadata, etc. Depending on the component, they might have their own database folder reserved just for them.

The database itself is an abstraction of a [Speedb](https://github.com/speedb-io/speedb) database - a simple key/value database, but handled in a different way: keys use *prefixes*, which makes it possible to batch read and write, so we can get around the "simple key/value" limitation and divide data into logical sectors.

For contract events specifically, there is a separate [SQLite](https://sqlite.org) abstraction used for better querying capabilities. The rest of this section is related to the Speedb part.

## General overview

The database requires a filesystem path to open it (if it already exists) or create it on the spot (if it doesn't exist) during construction. It closes itself automatically on destruction. Optionally, it also accepts a bool for enabling compression (disabled by default), if needed.

Content in the database is stored as raw bytes for space optimization purposes - one raw byte equals two UTF-8 characters (e.g. an address like `0x1234567890123456789012345678901234567890`, ignoring the "0x" prefix, occupies 20 raw bytes - "12 34 56 ..." - , but 40 bytes if converted to a string, since each byte becomes two separate characters - "1 2 3 4 5 6 ...").

The main **DB** class supports the typical CRUD operations, as well as a few helper functions for managing batched operations, keys and prefixes (see below). Due to how Speedb works internally, updating an entry is the same as inserting a different value in a key that already exists, effectively replacing the value that existed before (e.g. `put(oldKey, newValue)`).

There are three helper structs that further abstract database manipulation in general:

* **DBServer** - contains the host and version of the database that will be connected to
* **DBEntry** - abstraction of an entry to be inserted or read by the database, and has only two members: key and value, both strings
* **DBBatch** - abstraction for a list of multiple `DBEntry`s to be inserted and/or deleted all at once

There's also a **DBPrefix** namespace for referencing the database's prefixes in a simpler way, using labels instead of raw byte strings. Those prefixes are concatenated to the start of the *key*, so an entry that would have, for example, a key named "abc" and a value of "123", if inserted to the "0003" prefix, would be like this inside the database (in raw bytes, shown as strings here for explanation purposes): `{"0003abc": "123"}`.

## Prefixes overview

Here's a list of available prefixes from DBPrefix (they're accessed like `DBPrefix::label`, where you change `label` for one of the desired names below):

| Descriptor          | Prefix |
| ------------------- | ------ |
| blocks              | 0x0001 |
| heightToBlock       | 0x0002 |
| nativeAccounts      | 0x0003 |
| txToBlock           | 0x0004 |
| rdPoS               | 0x0005 |
| contracts           | 0x0006 |
| contractManager     | 0x0007 |
| events (DEPRECATED) | 0x0008 |
| vmStorage           | 0x0009 |
| txToAdditionalData  | 0x000A |
| txToCallTrace       | 0x000B |

### blocks

Used to store serialized blocks based on their hashes.

| Key                | Value            |
| ------------------ | ---------------- |
| Prefix + BlockHash | Serialized Block |

### heightToBlock

Used to store block hashes based on their heights.

| Key                                   | Value     |
| ------------------------------------- | --------- |
| Prefix + Padded uint64\_t BlockHeight | BlockHash |

Padded means that the `uint64_t` is padded with 0's to the left to ensure a fixed length of 8 bytes.

### nativeAccounts

Used to store serialized native accounts ("balance + nonce") based on their addresses.

| Key              | Value                    |
| ---------------- | ------------------------ |
| Prefix + Address | Serialized NativeAccount |

Serialization for a native account goes like this: `requiredBytes(balance) + bytes(balance) + requiredBytes(nonce) + bytes(nonce)`.

For example, an account with balance 1000000 and nonce 2 would be serialized as `03 + 0f4240 + 01 + 02`.

An account with balance 0 and nonce 0 would be serialized as `0000`.

### txToBlock

Used to store block hashes, the tx indexes within that block and the block heights, based on their transaction hashes.

| Key                      | Value                                                                  |
| ------------------------ | ---------------------------------------------------------------------- |
| Prefix + TransactionHash | BlockHash + Padded uint32\_t BlockIndex + Padded uint64\_t BlockHeight |

Padded means that the `uint32_t` and `uint64_t` are padded with 0's to the left to ensure a fixed length of 4 and 8 bytes respectively.

### rdPoS

Used to store Validator addresses based on their index within the rdPoS list.

| Key                                      | Value   |
| ---------------------------------------- | ------- |
| Prefix + Padded uint64\_t ValidatorIndex | Address |

### contracts

Used in a multitude of ways, where an "additional" prefix is used per contract (the contract address).

| Key                                           | Value                        |
| --------------------------------------------- | ---------------------------- |
| Prefix + Contract Address + "contractName"    | The Contract Class Name      |
| Prefix + Contract Address + "contractAddress  | The Contract Address         |
| Prefix + Contract Address + "contractCreator" | The Contract Creator Address |
| Prefix + Contract Address + "contractChainId" | The Contract ChainId         |

Contracts are free to use the contract prefix as they wish, but the following is the default structure for the contract prefix:

| Key                                                      | Value                                |
| -------------------------------------------------------- | ------------------------------------ |
| Prefix + Contract Address + "VARIABLE\_NAME\_AS\_STRING" | The Contract Variable to store in DB |

### contractManager

Used to store a contract class name based on their address.

| Key                       | Value               |
| ------------------------- | ------------------- |
| Prefix + Contract Address | Contract Class Name |

### events (DEPRECATED)

Previously used to store events emitted from contracts. Replaced by the SQLite abstraction. DO NOT USE.

| Key                                                                                                              | Value                                  |
| ---------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| Prefix + Padded uint64\_t Block Index + Padded uint64\_t Tx Index + Padded uint64\_t Log Index + ContractAddress | Event data serialized as a JSON string |

### vmStorage

Used to store EVM-related stuff like storage keys and values (essentially the EVM's "permanent memory").

| Key                                         | Value                    |
| ------------------------------------------- | ------------------------ |
| Address (20 bytes) + Storage Key (32 bytes) | Storage Value (32 bytes) |

### txToAdditionalData

Used to store EVM transactions that created contracts, storing the transaction hash and additional data about the contract that was deployed (see the `TxAdditionalData` struct in `utils/tx.h` for more details).

| Key                      | Value                              |
| ------------------------ | ---------------------------------- |
| Prefix + TransactionHash | Serialized TxAdditionalData struct |

### txToCallTrace

Used to store debugging information about contract calls - see the `Call` struct in `contract/calltracer.h` for more details.

| Key                      | Value                  |
| ------------------------ | ---------------------- |
| Prefix + TransactionHash | Serialized Call struct |
