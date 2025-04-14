---
description: How contract calls happen from both sides in AppLayer
---

# Handling contract calls

Here's a quick overview on how AppLayer differentiates between both types of contract calls (native and EVM). As always, we recommend reading the "BDK implementation" for more details on how it all works.

We employ templated functions to support flexible and efficient interaction. These templates allow passing any combination of arguments and return types (including `void`) to and from other types of contracts. This helps with leveraging a fast ABI encoding/decoding process, ensuring optimal performance and flexibility during contract execution and allowing for dynamic contract interactions by accommodating various contract behaviors and states, without having to pre-define all possible function signatures.

## Native calls

The `ContractHost` class employs several functions dedicated to native contract execution contexts. For calls from a native (e.g. C++) contract to another contract, we have two main templated functions - `callContractViewFunction()` (for view functions) and `callContractFunction()` (for non-view/callable/non-payable functions):

```cpp
template <typename R, typename C, typename... Args>
R ContractHost::callContractViewFunction(
  const BaseContract* caller, const Address& targetAddr,
  R(C::*func)(const Args&...) const, const Args&... args
) const;

template <typename R, typename C, typename... Args>
R ContractHost::callContractFunction(
  BaseContract* caller, const Address& targetAddr,
  const uint256_t& value,
  R(C::*func)(const Args&...), const Args&... args
);
```

## EVM calls

The `EvmContractExecutor` class employs several functions dedicated to EVM contract execution contexts. For calls from the EVM to another contract, the `call()` function plays a crucial role. It is tasked with creating and handling calls to other contracts, encapsulating the complexity of contract interaction within a simple interface.

This function is designed to handle all kinds of known EVM contract call types, as shown below:

```cpp
evmc::Result EvmContractExecutor::call(const evmc_message& msg) noexcept {
  Gas gas(msg.gas);
  const uint256_t value = EVMCConv::evmcUint256ToUint256(msg.value);

  const auto process = [&] (auto& msg) {
    try {
      const auto output = messageHandler_.onMessage(msg);

      if constexpr (concepts::CreateMessage<decltype(msg)>) {
        return evmc::Result(EVMC_SUCCESS, int64_t(gas), 0, bytes::cast<evmc_address>(output));
      } else {
        return evmc::Result(EVMC_SUCCESS, int64_t(gas), 0, output.data(), output.size());
      }
    } catch (const OutOfGas&) { // TODO: ExecutionReverted exception is important
      return evmc::Result(EVMC_OUT_OF_GAS);
    } catch (const std::exception& err) {
      Bytes output;

      if (err.what() != nullptr) {
        output = ABI::Encoder::encodeError(err.what()); // TODO: this may throw...
      }

      return evmc::Result(EVMC_REVERT, int64_t(gas), 0, output.data(), output.size());
    }
  };

  if (msg.kind == EVMC_DELEGATECALL) {
    EncodedDelegateCallMessage encodedMessage(msg.sender, msg.recipient, gas, value, View<Bytes>(msg.input_data, msg.input_size), msg.code_address);
    return process(encodedMessage);
  } else if (msg.kind == EVMC_CALL && msg.flags == EVMC_STATIC) {
    EncodedStaticCallMessage encodedMessage(msg.sender, msg.recipient, gas, View<Bytes>(msg.input_data, msg.input_size));
    return process(encodedMessage);
  } else if (msg.kind == EVMC_CALL) {
    EncodedCallMessage encodedMessage(msg.sender, msg.recipient, gas, value, View<Bytes>(msg.input_data, msg.input_size));
    return process(encodedMessage);
  } else if (msg.kind == EVMC_CREATE) {
    EncodedCreateMessage encodedMessage(msg.sender, gas, value, View<Bytes>(msg.input_data, msg.input_size));
    return process(encodedMessage);
  } else if (msg.kind == EVMC_CREATE2) {
    EncodedSaltCreateMessage encodedMessage(msg.sender, gas, value, View<Bytes>(msg.input_data, msg.input_size), msg.create2_salt);
    return process(encodedMessage);
  }

  std::unreachable();
}
```
