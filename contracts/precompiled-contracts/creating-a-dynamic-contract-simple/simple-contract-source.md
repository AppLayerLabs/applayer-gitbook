---
description: Coding the SimpleContract's source file
---

# Simple Contract Source

With the `SimpleContract` header, declarations and (most of the) registering done, now we can proceed to the contract's implementation.

## Defining the contract constructors and dump function

Open the source file (`simplecontract.cpp`) and `#include "simplecontract.h"` right at the beginning.

The first thing we'll implement is the constructors and dumping function of our contract class. The implementation must follow a certain order of events:

* The base `DynamicContract` constructor must be called and its respective arguments must be passed in order
* The contract's SafeVariables must be accessed with `this` (e.g. `this->name`) and initialized accordingly with their values if necessary (e.g. directly from the constructor, or by fetching values from the database)
* The contract's SafeVariables must call `commit()` to properly set their values to the values they were assigned during construction
* `registerContractFunctions()` must be called to properly register the contract's functions and events (if there are any)
* The contract's SafeVariables must call `enableRegister()` so they can be set to be properly marked as "used" during contract calls (this is required for the commit/revert logic to work)
* If anything happens during construction that would require throwing an exception, said throw should be done ***before*** calling `enableRegister()` on any SafeVariable - enabling registers should be the *last* thing done by the constructor to avoid heap-use-after-free errors caused by variables being accessed after a throw happens

Our source file will look something like this:

```cpp
#include "simplecontract.h"

#include "../../utils/uintconv.h"
#include "../../utils/strconv.h"
#include "../../utils/utils.h"

SimpleContract::SimpleContract(
  const std::string& name,
  const uint256_t& number,
  const std::tuple<std::string, uint256_t>& tuple,
  const Address& address,
  const Address& creator,
  const uint64_t& chainId
) : DynamicContract("SimpleContract", address, creator, chainId),
  name_(this), number_(this), tuple_(this)
{
  this->name_ = name;
  this->number_ = number;
  this->tuple_ = tuple;

  this->name_.commit();
  this->number_.commit();
  this->tuple_.commit();

  registerContractFunctions(); // DO NOT THROW AFTER THIS LINE!

  this->name_.enableRegister();
  this->number_.enableRegister();
  this->tuple_.enableRegister();
}

SimpleContract::SimpleContract(
  const Address& address,
  const DB& db
) : DynamicContract(address, db), name_(this), number_(this), tuple_(this) {
  this->name_ = StrConv::bytesToString(db.get(std::string("name_"), this->getDBPrefix()));
  this->number_ = UintConv::bytesToUint256(db.get(std::string("number_"), this->getDBPrefix()));
  this->tuple_ = std::make_tuple(
    StrConv::bytesToString(db.get(std::string("tuple_name"), this->getDBPrefix())),
    UintConv::bytesToUint256(db.get(std::string("tuple_number"), this->getDBPrefix()))
  );

  this->name_.commit();
  this->number_.commit();
  this->tuple_.commit();

  registerContractFunctions(); // DO NOT THROW AFTER THIS LINE!

  this->name_.enableRegister();
  this->number_.enableRegister();
  this->tuple_.enableRegister();
}

DBBatch SimpleContract::dump() const {
  DBBatch dbBatch = BaseContract::dump();
  dbBatch.push_back(StrConv::stringToBytes("name_"), StrConv::stringToBytes(this->name_.get()), this->getDBPrefix());
  dbBatch.push_back(StrConv::stringToBytes("number_"), UintConv::uint256ToBytes(this->number_.get()), this->getDBPrefix());
  dbBatch.push_back(StrConv::stringToBytes("tuple_name"), StrConv::stringToBytes(get<0>(this->tuple_)), this->getDBPrefix());
  dbBatch.push_back(StrConv::stringToBytes("tuple_number"), UintConv::uint256ToBytes(get<1>(this->tuple_)), this->getDBPrefix());
  return dbBatch;
}
```

Notice that, in the first constructor, we use `SimpleContract` as the `contractName` argument in the base `DynamicContract` constructor. As stated previously, this match is a **requirement**, otherwise it will result in a segfault. Both constructors initialize the inner variables of the contract - the first one using the arguments directly, and the second one loading them directly from the database.

The dumping function is called periodically and is responsible for collecting the contract variables' values and sending them back to `DumpManager` (the internal class that does the actual database dump, see "BDK implementation" for more details), which will save those values in the database so that they can be loaded later by the second constructor, when `ContractManager` is being constructed. `getDBPrefix()` is a getter for the contract's own prefix in the database, which would be equivalent to `DBPrefix::contracts` + the contract's address.

Keep in mind that **the database stores data as raw bytes** - this is why we use the respective conversion functions from Utils when saving (`XyzToBytes()`) and loading (`bytesToXyz()`) variables.

Also keep in mind that **you should always dump every parent contract class' data** as well - this is why we do `BaseContract::dump()` in the code above, since Dynamic Contracts inherit directly from `BaseContract` and their metadata (creator, timestamp, etc.) is stored there. Another example would be the `NativeWrapper` contract - it inherits directly from the `ERC20` class and depends on its variables to construct itself, so you must dump *both* (see `src/contract/templates/nativewrapper.cpp` for more details). Forgetting to do this may result in undefined behaviour when loading the contract's data from the database.

## Defining contract functions

Now let's implement the proper functions of our contract - first, the **view** functions (that only read and never change the contract's variables when called), then, the **non-view** functions (that do change the contract's variables when called).

**View** functions MUST be `const`, while **non-view** functions MUST NOT be `const`, and both functions can return either `void` or one of the ABI-supported types.

In our case, we have three view functions which would be `getName()`, `getNumber()` and `getTuple()`, which are the getters for the variables of our contract - `name_`, `number_` and `tuple_`, respectively. We can return the inner data from any SafeVariable by calling the `get()` function, like this:

```cpp
std::string SimpleContract::getName() const { return this->name_.get(); }
uint256_t SimpleContract::getNumber() const { return this->number_.get(); }
std::tuple<std::string, uint256_t> SimpleContract::getTuple() const {
  return std::make_tuple(get<0>(this->tuple_), get<1>(this->tuple_));
}
```

Note the templated `get<>()` functions in the tuple are *not* C++'s `std::get<>()` implementation, but rather the SafeTuple's own implementation. This is due to how SafeVariables work internally - as it is custom functionality, the C++ Standard Library is unaware of it, thus using `std::` here is not going to work (or even compile for that matter).

For the three non-view functions we have, which would be the setters (`setName()`, `setNumber()` and `setTuple()` respectively), we must also check that whoever is calling those functions is the actual creator of the contract, as we want to prevent unwanted calls from other addresses (this is how it's coded in the original Solidity code reference). We can do that by calling `getCaller()` and `getContractCreator()`, respectively, to access the address of the caller and the address of the contract creator, and then we check if both addresses are the same.

If your contract has events, you can emit them by simply calling them like they were any other function (they actually are if you think about it!). The only thing you have to pay attention to is that **events can ONLY be emitted from NON-view functions**, due to how const correctness works in C++ (view functions are `const`, so trying to emit an event from one of them will result in a compilation error).

```cpp
void SimpleContract::setName(const std::string& argName) {
  if (this->getCaller() != this->getContractCreator()) {
    throw std::runtime_error("Only contract creator can call this function.");
  }
  this->name_ = argName;
  this->nameChanged(this->name_.get());
}

void SimpleContract::setNumber(uint256_t argNumber) {
  if (this->getCaller() != this->getContractCreator()) {
    throw std::runtime_error("Only contract creator can call this function.");
  }
  this->number_ = argNumber;
  this->numberChanged(this->number_.get());
}

void SimpleContract::setTuple(const std::tuple<std::string, uint256_t>& argTuple) {
  if (this->getCaller() != this->getContractCreator()) {
    throw std::runtime_error("Only contract creator can call this function.");
  }
  this->tuple_ = argTuple;
  this->tupleChanged(std::make_tuple(get<0>(this->tuple_), get<1>(this->tuple_)));
}
```

## Registering contract functions

After all functions are implemented, we must implement one more - `registerContractFunctions()`, which is responsible for registering the other functions so they can be called later by a transaction or an RPC `eth_call`. Their respective functors/signatures will be stored in an internal map, allowing a given transaction to call any function within that contract. Registration is done within try/catch blocks internally, which allows the protection of SafeVariables against any exceptions thrown by the function.

The first thing it should do is call `registerContract()` right away, so it's guaranteed that the contract itself will be registered before its functions. As for the functions themselves, they are registered by calling `this->registerMemberFunctions()` and passing to it several tuples - one for each function your contract has (NOT including events). Each tuple needs four arguments - the function's name, a reference to the function, its state mutability, and `this` (a pointer to the contract itself), as follows:

```cpp
void SimpleContract::registerContractFunctions() {
  registerContract();
  this->registerMemberFunctions(
    std::make_tuple("getName", &SimpleContract::getName, FunctionTypes::View, this),
    std::make_tuple("getNumber", &SimpleContract::getNumber, FunctionTypes::View, this),
    std::make_tuple("getTuple", &SimpleContract::getTuple, FunctionTypes::View, this),
    std::make_tuple("setName", &SimpleContract::setName, FunctionTypes::NonPayable, this),
    std::make_tuple("setNumber", &SimpleContract::setNumber, FunctionTypes::NonPayable, this),
    std::make_tuple("setTuple", &SimpleContract::setTuple, FunctionTypes::NonPayable, this)
  );
}
```

Note that the complete implementation of this contract has overloads on `getNumber()`, so you may see `static_cast<uint256_t(SimpleContract::*)() const>(&SimpleContract::getNumber)` in place of simply `&SimpleContract::getNumber` - it's done this way so we know exactly which function we are referring to. Since this implementation is a simplified version and we only have one `getNumber()` function in it, we reference it here simply as `&SimpleContract::getNumber`. If your contract has one or more overloads for the same function though, you should keep this in mind and cast them accordingly.
