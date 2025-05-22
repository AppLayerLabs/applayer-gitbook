---
description: A primer on natively-coded smart contracts in the AppLayer ecosystem
---

# Precompiled contracts

Precompiled contracts (also known as "native contracts", "natively-coded contracts" or "stateful pre-compiles") are contracts coded with the blockchain's native language. Things like transaction parsing methods and arguments, as well as management and storage of the contract's variables in a database, are manually coded in the blockchain's native language (e.g. C++ in bdk-cpp) to be tightly integrated with the blockchain itself.

The term "stateful pre-compile" comes from the notion that it can maintain a state (thus "stateful") and is basically machine code, not interpreted by a virtual machine (thus "pre-compile").

Similar to Solidity contracts, they can be used to employ any type of logic within the network. Unlike Solidity, however, they aren’t subject to EVM constraints. This means we can take advantage of that fact and have full control of the contract's logic, unleashing blazing fast performance, flexibility and power.

The precompiled contract templates provided by AppLayer's BDK (in the `src/contract/templates` folder) are based on OpenZeppelin contracts, maintaining the same operational standards known in the Solidity ecosystem, but coded in C++ (in the case of bdk-cpp).

## Example of a precompiled contract

Given the example Solidity contract:

```cpp
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.10;

contract ExampleContract {
    mapping(address => uint256) values;
    function setValue(address addr, uint256 value) external {
        values[addr] = value;
        return;
    }
}
```

As a precompiled C++ contract, the code should look something like this (don't worry about the details, they will be explained in further subchapters):

### Contract header

```cpp
#include <...>
class ExampleContract : public DynamicContract {
  private:
    std::unordered_map<Address, uint256_t> values; // or boost::unordered_flat_map for example
    // Const-reference as they are not changed by the function.
    void setValue(const Address& addr, const uint256_t& value);
  public:
    // Constructor for creating the contract the first time.
    // "address" is where the contract will be deployed.
    // "creator" is the address that created the contract.
    // "chainId" is the chain ID where the contract will operate.
    ExampleContract(
      const Address& address, const Address& creator, const uint64_t& chainId
    );
    // Constructor for loading the already-created contract from the database.
    ExampleContract(
      const Address& address, const DB& db
    );
    void callContractWithTransaction(const Tx& transaction);
}
```

### Contract source

```cpp
#include "ExampleContract.h"

ExampleContract(
  const Address& address, const Address& creator, const uint64_t& chainId
) : DynamicContract("ExampleContract", address, creator, chainId) {
  // Initialize the contract's variables for the first time as the contract is being created
  ...
}

ExampleContract::ExampleContract(
  const Address& address,
  const DB& db
) : DynamicContract(address, db) {
  // Load the contract's variables from the database as it already exists there
  ...
}

void ExampleContract::setValue(const Address &addr, const uint256 &value) {
  this->values[addr] = value;
  return;
}

void ExampleContract::callContractWithTransaction(const Tx& transaction) {
  // Used to route and decode transactions.
  // The data inside the if block is only an example.
  // A real contract would match both functor and arguments to the called contract.
  std::string_view txData = transaction.getData();
  auto functor = txData.substr(0,8);
  // "0x48461b56" is equivalent to Keccak256("setValue(address,uint256)")
  if (functor == Utils::hexToBytes("0x48461b56")) {
    this->setValue(ABI::Decoder::decodeAddress(txData, 8), ABI::Decoder::decodeUint256(txData, 8 + 32));
  }
  return;
}
```
