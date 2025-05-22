---
description: How different types of contracts call each other in AppLayer
---

# Calling contracts

The previous subchapters explained how both types of contracts (native and EVM) are handled and executed internally in the BDK. Here's how they actually interact with each other in AppLayer's ecosystem (as in, at a higher "user/developer" level of abstraction).

Most details from native contract implementations were taken out for simplicity purposes. See the "Precompiled contracts" subsection for more details on how to properly code native contracts.

## Calling native contracts from EVM

To invoke a native (e.g. C++) contract by calling it from an EVM contract, we use the standard Solidity interface to abstract the native implementation. This approach ensures that EVM-to-native calls are as straightforward as EVM-to-EVM calls.

For example, let's suppose we have a C++ contract coded like this:

```c++
class MyContract : public DynamicContract {
  private:
    // ...some code...
  public:
    // ...some more code...
    uint256_t myFunction(const uint256_t& arg1, const uint256_t& arg2) const;
    // ...yet some more code...
}
```

First, we define a Solidity interface that matches the signature of the native functions you wish to call. This interface acts as a facade, providing a Solidity view of the naitive contract's functionalities:

```solidity
interface MyContract {
  // ...some code...
  function myFunction(uint256 arg1, uint256 arg2) external view returns (uint256);
  // ...some more code...
}
```

Then, we use the defined interface to make calls to the native contract. This is handled similarly to any inter-contract communication in Solidity, ensuring a seamless integration layer:

```solidity
contract AnotherContract {
  function callMyFunction(address cppAddr, uint256 arg1, uint256 arg2) public view returns (uint256) {
    return MyContract(cppAddr).myFunction(arg1, arg2);
  }
}
```

## Calling EVM contracts from native

To invoke an EVM contract by calling it from a native contract, we leverage a templated approach that mimics the EVM contract's functions in a native class (basically the same idea as above but going the other way around). This approach provides a type-safe way to interact with contracts written in Solidity or other EVM-compatible languages.

For example, let's suppose we have the same contract from the previous section, coded in Solidity:

```solidity
interface MyContract {
  // ...some code...
  function myFunction(uint256 arg1, uint256 arg2) external view returns (uint256);
  // ...some more code...
}
```

First, we define a proxy native (in this case, C++) class that represents the EVM contract. This class will include stubs of the contract's functions, which do not contain actual logic but serve to match the contract's interface in the blockchain:

```cpp
class MyContract {
  private:
    // ...some code...
  public:
    // ...some more code...
    uint256_t myFunction(const uint256_t& arg1, const uint256_t& arg2) const;
    // ...yet some more code...
};
```

Then, we ensure that the proxy class is registered within the blockchain before any calls are made. Typically, this registration is done once, often in the constructor of the calling native contract (using `registerContract()`), to set up the reflection system used for method invocation:

```cpp
uint256_t AnotherContract::callMyFunction(const Address& targetAddr, const uint256_t& arg1, const uint256_t& arg2) const {
  MyContract::registerContract(); // This is usually done in the contract itself
  return this->callContractViewFunction<MyContract>(this, targetAddr, &MyContract::myFunction, arg1, arg2);
}
```
