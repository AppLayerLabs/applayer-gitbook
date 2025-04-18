---
description: How the BDK uses RLP to encode and decode transactions
---

# RLP (Recursive-Length Prefix)

Transactions coming from the network are (de)serialized through a standard called [Recursive-Length Prefix](http://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp), also known as **RLP** and used extensively by Ethereum. As per their definition:

*"RLP standardizes the transfer of data between nodes in a space-efficient-format. The only purpose of RLP is to encode structure and arbitrarily nested arrays of binary data."*

## Encoding/Decoding rules

First of all, **the first byte of the data string defines what exactly the serialized string is storing**. This can be broken down as follows:

* **A**: `byte < 0x80` means the value is the byte itself
  * e.g. `0x55` means the value is exactly `0x55`
* **B**: `0x80 > byte < 0xb7` means the value is between 0 and 55 bytes, calculated by subtracting `0x80` from it
  * e.g. `0x90` means the next 16 bytes after `0x90` contains the payload for that index (`0x90 - 0x80 = 0x10 -> 16 bytes in decimal`)
  * `0x900102030405060708090a0b0c0d0e0f10` -> value is the next 16 bytes after `0x90`, which would be `0x0102030405060708090a0b0c0d0e0f10`
* **C**: `0xb7 > byte < 0xc0` means the value is longer than 55 bytes, calculated by subtracting `0xb7` from it
  * e.g. `0xb9012c4e6bffdadac55ac51ec4718aeb6ea75db805d673c3b98128ff02558106170b7e66f38c573131bc0a1a0826fa90b7063a66a6a5f9a793d2295f874769349ffda6fc30e9154a787dbd604524497214cf97f1b56997107429718f6cc92804698972664af8a8c782da4f32a83ce20db38bf09f01a662fa598435a80780912daaa7e4a37be2feabcb90de0b717128c199cf8d18e31fa7bffd`
  * `0xb9 - 0xb7 = 0x02 -> 2 bytes in decimal` -> size is the next 2 bytes after `0xb9`, which would be `0x012c = 300 bytes` -> value is the next 300 bytes after `0x012c`
* **D**: `byte > 0xc0` means the value is a list of 0 to 55 bytes, calculated by subtracting `0xc0` from it
  * e.g. `0xc50000000000` -> `0xc5 - 0xc0 = 0x05 -> 5 bytes in decimal` -> value is the next 5 bytes after `0xc5`, which would be `0x0000000000`
  * This is not a valid list, as we don't use lists lower than 55 bytes when deserializing a transaction, since transactions are always larger than 55 bytes
* **E**: `byte > 0xf7` means the value is a list longer than 55 bytes, calculated by subtracting `0xf7` from it
  * e.g. `0xf8a90c8504a817c80082c160944fabb145d64652a948d72533023f6e7a623c7c5380b844a9059cbb0000000000000000000000006b71dcaa3fb9a4901491b748074a314dad9e980b000000000000000000000000000000000000000000000029e7ab336ae0b5000025a0ef2f3450e6860289dce618af68ebc7d518c3cb3ea4d1641cb2fe7c7251ff31d4a0540dcf1500630a1b0d0d0670eee012e2cf2c64cf3288d122e0efb0d3deb0340f`
  * `0xf8 - 0xf7 = 0x01 -> 1 byte in decimal` -> size is the next 1 byte after `0xf7`, which would be `0xa9 = 169 bytes` -> value is the next 169 bytes after `0xa9`

## Decoding a transaction

Let's take an example of a [serialized and signed transaction](https://etherscan.io/tx/0xfd394cb193386ae904af2ef19247e16c51e6974aa8505dbc9b699cc289fb180d) and decode it:

`f8a90c8504a817c80082c160944fabb145d64652a948d72533023f6e7a623c7c5380b844a9059cbb0000000000000000000000006b71dcaa3fb9a4901491b748074a314dad9e980b000000000000000000000000000000000000000000000029e7ab336ae0b5000025a0ef2f3450e6860289dce618af68ebc7d518c3cb3ea4d1641cb2fe7c7251ff31d4a0540dcf1500630a1b0d0d0670eee012e2cf2c64cf3288d122e0efb0d3deb0340f`

* We process the first byte: `0xf8` - this tells us the RLP payload is a list
* `0xf8 - 0xf7 = 0x01 -> 1 byte` - the size of the payload is the next 1 byte (`0xa9`)
* `0xa9 = 169 bytes` -> the payload is the next 169 bytes

When serializing a transaction to the database, we also append the `from` address to the end of the string, OUTSIDE of the list so it won't mess up the transaction data. Keep that in mind when deserializing from the database.

From here, we'll decode the rest of the data, in this order (following after `0xf8a9`):

**1 - Nonce** -> can be either a single byte (rule A) or a payload under 55 bytes (rule B)

* `0x0c` -> less than `0x80`, thus rule A -> nonce is `0x0c = 12 in decimal`

**2 - Gas Price** -> always a payload under 55 bytes (rule B)

* `0x85` -> `0x85 - 0x80 = 0x05 -> 5 bytes` -> gas price is `0x04a817c800 = 20000000000 in decimal (uint256)`

**3 - Gas Limit** -> always a payload under 55 bytes (rule B), because the minimum gas limit for an Ethereum transaction is 21000

* `0x82` -> `0x82 - 0x80 = 0x02 -> 2 bytes` -> gas limit is `0xc160 = 49504 in decimal (uint256)`

**4 - Receiver address ("to")** -> always a payload under 55 bytes (rule B), because we know an address is _always_ 20 bytes

* `0x94` -> `0x94 - 0x80 = 0x14 -> 20 bytes` -> receiver address is `0x4fabb145d64652a948d72533023f6e7a623c7c53`

**5 - Value** -> always a payload under 55 bytes (rule B)

* `0x80` -> `0x80 - 0x80 = 0 bytes` -> value is 0

**6 - Arbitrary data** -> can be either a value between 0 and 55 (rule B) or longer (rule C)

* `0xb8` -> more than `0xb7`, thus rule C -> `0xb8 - 0xb7 = 1 byte` -> size is `0x44 = 68 bytes` -> data is `0xa9059cbb0000000000000000000000006b71dcaa3fb9a4901491b748074a314dad9e980b000000000000000000000000000000000000000000000029e7ab336ae0b50000`

**7 - ECDSA signature recovery ID ("v")** -> can be either a single byte (rule A) or a payload under 55 bytes (rule B)

* `0x25` -> less than `0x80`, thus rule A -> v is `0x25 = 37 in decimal`

**8 - ECDSA signature first half ("r")** -> always a payload under 55 bytes (rule B)

* `0xa0` - `0xa0 - 0x80 = 0x20 -> 32 bytes` -> r is `0xef2f3450e6860289dce618af68ebc7d518c3cb3ea4d1641cb2fe7c7251ff31d4`

**9 - ECDSA signature second half ("s")** -> always a payload under 55 bytes (rule B)

* `0xa0` - `0xa0 - 0x80 = 0x20 -> 32 bytes` -> s is `0x540dcf1500630a1b0d0d0670eee012e2cf2c64cf3288d122e0efb0d3deb0340f`
