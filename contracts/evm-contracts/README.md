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
  const Storage& storage,
  const Hash& randomnessSeed,
  const evmc_tx_context& currentTxContext,
  boost::unordered_flat_map<Address, std::unique_ptr<BaseContract>, SafeHash>& contracts,
  boost::unordered_flat_map<Address, NonNullUniquePtr<Account>, SafeHash>& accounts,
  boost::unordered_flat_map<StorageKey, Hash, SafeHash>& vmStorage,
  const Hash& txHash,
  const uint64_t txIndex,
  const Hash& blockHash,
  int64_t& txGasLimit
);
```

Once an instance of `ContractHost` is created, it offers methods like `execute()` to run the contract, `simulate()` for simulating the transaction (useful for gas estimation), and `ethCallView()` for making calls to other contracts within a non-state-changing context.

`ContractHost` also extends the functionalities of `evmc::Host` by overriding several key functions that interface directly with the Ethereum Virtual Machine, which are obligatory for the VM to be able to interact with the blockchain's state:

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

These methods manage everything from account validation to logging, providing access to the blockchain's state and storage, and handling calls between contracts. The `ContractHost` class encapsulates these functions, ensuring that each contract execution is properly secured and isolated from each other.

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

### Determining contract types and executing calls

`ContractHost` plays a critical role in distinguishing whether a contract is implemented natively or in EVM bytecode and executing it accordingly. Below is an example illustrating how contracts can invoke functions in other contracts, whether they are coded in e.g. C++ or Solidity:

```c++
template <typename R, typename C, typename... Args>
R callContractFunctionImpl(
  BaseContract* caller, const Address& targetAddr,
  const uint256_t& value,
  R(C::*func)(const Args&...), const Args&... args
) {
  auto& recipientAcc = *this->accounts_[targetAddr];
  if (!recipientAcc.isContract()) {
    throw DynamicException(std::string(__func__) + ": Contract does not exist - Type: "
      + Utils::getRealTypeName<C>() + " at address: " + targetAddr.hex().get()
    );
  }
  if (value) {
    this->sendTokens(caller, targetAddr, value);
  }
  NestedCallSafeGuard guard(caller, caller->caller_, caller->value_);
  switch (recipientAcc.contractType) {
    case ContractType::EVM: {
      this->deduceGas(10000);
      evmc_message msg;
      msg.kind = EVMC_CALL;
      msg.flags = 0;
      msg.depth = 1;
      msg.gas = this->leftoverGas_;
      msg.recipient = targetAddr.toEvmcAddress();
      msg.sender = caller->getContractAddress().toEvmcAddress();
      auto functionName = ContractReflectionInterface::getFunctionName(func);
      if (functionName.empty()) {
        throw DynamicException("ContractHost::callContractFunction: EVM contract function name is empty (contract not registered?)");
      }
      auto functor = ABI::FunctorEncoder::encode<Args...>(functionName);
      Bytes fullData;
      Utils::appendBytes(fullData, UintConv::uint32ToBytes(functor.value));
      if constexpr (sizeof...(Args) > 0) {
        Utils::appendBytes(fullData, ABI::Encoder::encodeData<Args...>(args...));
      }
      msg.input_data = fullData.data();
      msg.input_size = fullData.size();
      msg.value = EVMCConv::uint256ToEvmcUint256(value);
      msg.create2_salt = {};
      msg.code_address = targetAddr.toEvmcAddress();
      evmc::Result result (evmc_execute(this->vm_, &this->get_interface(), this->to_context(),
      evmc_revision::EVMC_LATEST_STABLE_REVISION, &msg, recipientAcc.code.data(), recipientAcc.code.size()));
      this->leftoverGas_ = result.gas_left;
      if (result.status_code) {
        auto hexResult = Hex::fromBytes(bytes::View(result.output_data, result.output_data + result.output_size));
        throw DynamicException("ContractHost::callContractFunction: EVMC call failed - Type: "
          + Utils::getRealTypeName<C>() + " at address: " + targetAddr.hex().get() + " - Result: " + hexResult.get()
        );
      }
      if constexpr (std::same_as<R, void>) {
        return;
      } else {
        return std::get<0>(ABI::Decoder::decodeData<R>(bytes::View(result.output_data, result.output_data + result.output_size)));
      }
    } break;
    case ContractType::CPP: {
      this->deduceGas(1000);
      C* contract = this->getContract<C>(targetAddr);
      this->setContractVars(contract, caller->getContractAddress(), value);
      try {
        return contract->callContractFunction(this, func, args...);
      } catch (const std::exception& e) {
        throw DynamicException(e.what() + std::string(" - Type: ")
          + Utils::getRealTypeName<C>() + " at address: " + targetAddr.hex().get()
        );
      }
    }
    default: {
      throw DynamicException("PANIC! ContractHost::callContractFunction: Unknown contract type");
    }
  }
}
```
