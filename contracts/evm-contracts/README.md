---
description: A primer on how EVM smart contracts work in the AppLayer ecosystem
---

# EVM contracts

Aside from native contracts, AppLayer can also execute Solidity contracts as-is by the use of the AppLayer EVM, which is compatible with bytecode deployment. This means any language that compiles to EVM bytecode (e.g. [Solidity](https://soliditylang.org/), [Vyper](https://docs.vyperlang.org/en/stable/), etc.) can be used to deploy contracts in the AppLayer EVM in a seamless, straight-forward way.

This kind of compatibility is possible thanks to the integration of the [EVMOne](https://github.com/ethereum/evmone) virtual machine (originally made by the Ethereum devs) and [EVMC](https://github.com/ethereum/evmc) libraries. Read the "BDK implementation" section for more details.

## State management and VM instance creation

The VM itself is owned and instantiated by the `State` class, which reflects a crucial design decision: centralizing the management of virtual machine resources ensures that each contract execution context is cleanly managed and isolated. Whenever a new transaction or contract call needs to be executed, regardless of its nature (be it a contract execution or a simple native transfer), the `State` class is responsible for instantiating a new `ContractHost` object with the relevant parameters required for execution:

```cpp
ContractHost(
  evmc_vm* vm,
  DumpManager& manager,
  Storage& storage,
  const Hash& randomnessSeed,
  ExecutionContext& context,
  BlockObservers *blockObservers = nullptr
);
```

### Determining contract types and executing calls

Once an instance of `ContractHost` is created, it delegates contract execution calls to a few members like `ExecutionContext` (which keeps track of data like transaction hash, index, gas limit, etc., as well as the logic required for reverting alterations made during the call when required, e.g. when it fails), `MessageDispatcher` (which re-routes the call to its respective executor - `CppContractExecutor` for C++ calls and `EvmContractExecutor` for EVM calls) and `CallTracer` (which answers RPC calls related to debugging purposes, if the node is set to RPC_TRACE).

`EvmContractExecutor` also extends the functionalities of `evmc::Host` by overriding several key functions that interface directly with the Ethereum Virtual Machine, which are obligatory for the VM to be able to interact with the blockchain's state:

```cpp
bool account_exists(const evmc::address& addr) const noexcept final;
evmc::bytes32 get_storage(const evmc::address& addr, const evmc::bytes32& key) const noexcept final;
evmc_storage_status set_storage(const evmc::address& addr, const evmc::bytes32& key, const evmc::bytes32& value) noexcept final;
evmc::uint256be get_balance(const evmc::address& addr) const noexcept final;
size_t get_code_size(const evmc::address& addr) const noexcept final;
evmc::bytes32 get_code_hash(const evmc::address& addr) const noexcept final;
size_t copy_code(const evmc::address& addr, size_t code_offset, uint8_t* buffer_data, size_t buffer_size) const noexcept final;
bool selfdestruct(const evmc::address& addr, const evmc::address& beneficiary) noexcept final;
evmc::Result call(const evmc_message& msg) noexcept final;
evmc_tx_context get_tx_context() const noexcept final;
evmc::bytes32 get_block_hash(int64_t number) const noexcept final;
void emit_log(const evmc::address& addr, const uint8_t* data, size_t data_size, const evmc::bytes32 topics[], size_t topics_count) noexcept final;
evmc_access_status access_account(const evmc::address& addr) noexcept final;
evmc_access_status access_storage(const evmc::address& addr, const evmc::bytes32& key) noexcept final;
evmc::bytes32 get_transient_storage(const evmc::address &addr, const evmc::bytes32 &key) const noexcept final;
void set_transient_storage(const evmc::address &addr, const evmc::bytes32 &key, const evmc::bytes32 &value) noexcept final;
```

These methods manage everything from account validation to logging, providing access to the blockchain's state and storage, and handling calls between contracts. The `EvmContractExecutor` class encapsulates these functions, ensuring that each contract execution is properly secured and isolated from each other.

## Seamless native/EVM integration

Achieving seamless integration between native (e.g. C++) and EVM contracts revolves around the uniformity in the encoding and decoding of their arguments. By standardizing the process, we ensure that calls between different contract types are handled efficiently without the need for separate mechanisms, therefore allowing the BDK to handle it all at once.

### The evmc\_message struct

We do this by using the `evmc_message` struct as a base, aligning the call structures between native and EVM environments. This uniformity simplifies the interaction framework and reduces the potential for errors and data mismanagement:

```cpp
struct evmc_message {
  enum evmc_call_kind kind;
  uint32_t flags;
  int32_t depth;
  int64_t gas;
  evmc_address recipient;
  evmc_address sender;
  const uint8_t* input_data;
  size_t input_size;
  evmc_uint256be value;
  evmc_bytes32 create2_salt;
  evmc_address code_address;
};
```
