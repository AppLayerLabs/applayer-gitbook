---
description: Coding the SimpleContract's header file
---

# Simple Contract Header

Having created the `SimpleContract` files and registered them in CMake, let's implement the contract's header first, as most of the registration is done there.

## Declaring the contract class

Open the header file (`simplecontract.h`) and add the following lines:

```cpp
#ifndef SIMPLECONTRACT_H
#define SIMPLECONTRACT_H

#include "../dynamiccontract.h"
#include "../variables/safestring.h"
#include "../variables/safetuple.h"
#include "../variables/safeuint.h"

class SimpleContract : public DynamicContract {
  private:
    // ...
  public:
    // ...
};

#endif // SIMPLECONTRACT_H
```

This is a simple skeleton so we can start building the proper contract. From top to bottom:

* We create [include guards](https://en.wikipedia.org/wiki/Include\_guard) as a safety measure
* We include the `DynamicContract` class and three SafeVariable classes for the contract's inner variables - `SafeString`, `SafeUint256_t` and `SafeTuple`, which represent a Solidity `string`, `uint256` and `tuple/struct` types, respectively (check the `src/contract/variables` subfolder for all available variable abstractions) - for SafeUint (and SafeInt as well), you can either use `SafeUint_t<X>`/`SafeInt_t<X>` or the `SafeUintX_t`/`SafeIntX_t` aliases directly, as they are included in their respective header files
* We create our `SimpleContract` class, inherit it from `DynamicContract`, and leave some space for the private and public members that will be coded next

Now we can declare the members of our contract. We have to pay attention to some rules described earlier, which we'll go through slowly, one part at a time.

## Declaring contract variables

Variables MUST be `private` and MUST inherit one of the SafeVariable classes (so the blockchain state can be aware of them). Our class declaration would start with something like this:

```cpp
class SimpleContract : public DynamicContract {
  private:
    SafeString name_;
    SafeUint256_t number_;
    SafeTuple<std::string, uint256_t> tuple_;
  // ...
}
```

Our three variables `name_`, `number_` and `tuple_` are respectively declared as a `SafeString`, `SafeUint256_t` and `SafeTuple<std::string, uint256_t>`. Notice we can use primitive types inside the tuple just fine, as the SafeVariable type itself ensures the necessary commit/revert safety of its children types.

## Declaring contract functions

Functions in general can return either `void` or an ABI-supported C++ type (see the correlations on the "Solidity ABI" section). The main difference is that **view** functions MUST be `const` (e.g. `getName() const;`), while **non-view** functions MUST NOT be `const` (e.g. `setName(std::string name);`).

For the two registering functions, `registerContract()` MUST be `public static void`, and `registerContractFunctions()` MUST be `private void override`. For the `dump()` function, it MUST be `const override` and return a `DBBatch` object:

```cpp
class SimpleContract : public DynamicContract {
  private:
    SafeString name_;
    SafeUint256_t number_;
    SafeTuple<std::string, uint256_t> tuple_;
    void registerContractFunctions() override;

  public:
    std::string getName() const;
    uint256_t getNumber() const;
    std::tuple<std::string, uint256_t> getTuple() const;
    void setName(const std::string& argName);
    void setNumber(uint256_t argNumber);
    void setTuple(const std::tuple<std::string, uint256_t>& argTuple);

    static void registerContract() {
      ...
    }

    DBBatch dump() const override;
};
```

Just like with the tuple variable, we can use primitive types and returns on functions just fine, for the same reasons stated above. This also extends to constructors, whch we'll see below.

## Declaring contract constructors

Like any C++ derived class, we must call its base class constructor and pass the proper arguments to it (besides the arguments for the derived class itself) so it can be constructed properly. Any contract derived from `DynamicContract` MUST have *two* constructors - one for creating a new contract from scratch, and another for loading the contract from the database:

```cpp
class SimpleContract : public DynamicContract {
  // ...
  public:
    // Constructor from scratch. Create new contract with given name and value.
    SimpleContract(
      const std::string& name,
      uint256_t number,
      const std::tuple<std::string, uint256_t>& tuple,
      const Address& address,
      const Address& creator,
      const uint64_t& chainId
    );

    // Constructor from load. Load contract from database.
    SimpleContract(
      const Address& address,
      const DB& db
    );

    // ...
}
```

Aside from our `name`, `number` and `tuple` variables, both constructors also take a few other arguments required by the base DynamicContract constructor:

```cpp
// Constructor that creates the contract from scratch
DynamicContract(
  const Address& address,
  const Address& creator,
  const uint64_t& chainId
) : BaseContract("SimpleContract", address, creator, chainId) {};

// Constructor that loads the contract from the database
DynamicContract(
  const Address& address,
  const DB& db
) : BaseContract(address, db) {};
```

`address`, `creator`, `chainId` and `db` are internal variables used by the base class, and should *always* be declared *last* (as in, *after* the contract's own variables if they exist). They are equivalent to:

| DynamicContract constructor argument | Taken from                                  |
| ------------------------------------ | ------------------------------------------- |
| address                              | derived using this->deriveContractAddress() |
| creator                              | this->getCaller()                           |
| chainId                              | this->options->getChainID()                 |
| DB                                   | this->db                                    |

Keep in mind that, when defining the base class constructor later on in the `.cpp` file, the contract's name argument (in this case, "SimpleContract") MUST be EXACTLY the same as your contract's class name. This is because the name is used to load the contract type from the database, so incorrectly naming it will result in a segfault at load time.

## Declaring contract events

If your contract has events, they MUST be `public`, `void`, non-`const`, AND call an internal function named `emitEvent()`, which is available to all Dynamic Contracts and does the proper emission of the event:

```cpp
class SimpleContract : public DynamicContract {
  private:
    SafeString name_; ///< The name of the contract.
    SafeUint256_t number_; ///< The number of the contract.
    void registerContractFunctions() override; ///< Register the contract functions.

  public:
    /// Event for when the name changes.
    void nameChanged(const EventParam<std::string, true>& name) {
      this->emitEvent(__func__, std::make_tuple(name));
    }

    /// Event for when the number changes.
    void numberChanged(const EventParam<uint256_t, false>& number) {
      this->emitEvent(__func__, std::make_tuple(number));
    }

    /// Event for when the name and number tuple changes.
    void tupleChanged(const EventParam<std::tuple<std::string, uint256_t>, true>& tuple) {
      this->emitEvent(__func__, std::make_tuple(tuple));
    }

  // ...
}
```

`emitEvent()` requires at most three arguments:

* The event name (you can just pass `__func__` which is the same as e.g. `this->emitEvent("nameChanged", ...)`)
* **(Optional)** A tuple of `EventParam` objects representing the event's arguments, where:
  * The first element is the argument type (e.g. `name` is a `std::string`, `number` is a `uint256_t`, `tuple` is a `std::tuple<std::string, uint256_t>`)
  * The second element is a bool that indicates whether the argument should be indexed or not
  * If your event has no arguments at all you can omit it or pass an empty tuple instead (e.g. `this->emitEvent(__func__, std::make_tuple())`)
* **(Optional)** A flag that indicates whether the event is anonymous or not
  * Events are non-anonymous by default (the flag defaults to `false`), so if you wish you can omit this (which is the case for our example, it is equivalent to `this->emitEvent(__func__, std::make_tuple(name), false)` - if it was an anonymous event, the last flag would be `true` instead)

## Registering the contract class

One last thing we have to do within our header is properly register the contract class. All Dynamic Contracts use templating to automate most of the hard work for both our contract and the BDK.

First, define a `public` tuple called `ConstructorArguments` with the contract's variable types from its first constructor (the one from scratch - in our example, name is `const std::string&`, number is `const uint256_t&`, tuple is `const std::tuple<std::string, uint256_t>&`):

```cpp
class SimpleContract : public DynamicContract {
  // ...
  public:
    using ConstructorArguments = std::tuple<
      const std::string&, const uint256_t&, const std::tuple<std::string, uint256_t>&
    >;
    // ...
}
```

Then, implement the `registerContract()` function declared previously by calling a function from the `DynamicContract` class called `registerContractMethods()`, passing a few arguments to it, like this:

```cpp
class SimpleContract : public DynamicContract {
  public:
    // ...
    static void registerContract() {
      static std::once_flag once;
      std::call_once(once, []() {
        DynamicContract::registerContractMethods<SimpleContract>(
          std::vector<std::string>{"name_", "number_", "tuple_"},
          std::make_tuple("getName", &SimpleContract::getName, FunctionTypes::View
, std::vector<std::string>{}),
          std::make_tuple("getNumber", &SimpleContract::getNumber, FunctionTypes::View
, std::vector<std::string>{}),
          std::make_tuple("getTuple", &SimpleContract::getTuple, FunctionTypes::View, std::vector<std::string>{}),
          std::make_tuple("setName", &SimpleContract::setName, FunctionTypes::NonPayable
, std::vector<std::string>{"argName"}),
          std::make_tuple("setNumber", &SimpleContract::setNumber, FunctionTypes::NonPayable
, std::vector<std::string>{"argNumber"}),
          std::make_tuple("setTuple", &SimpleContract::setTuple, FunctionTypes::NonPayable, std::vector<std::string>{"argTuple"})
        );
        // ...
      });
    }
}
```

The `std::call_once` function is used to guarantee contract registering will be done exactly once, even if called from several threads. Inside the chevrons (`registerContractMethods<...>()`) there's only one argument needed, which is the contract's class type itself (in this case, `SimpleContract` - *no quotes ""*).

Inside the parentheses (`registerContractMethods<>(...)`):

* The first argument is a string vector that is a list of all the exact names of the arguments in the constructor, each one separated by a comma - in this case, `"name_"`, `"number_"` and `"tuple_"`. If the contract has no variables in it, you can pass an empty list instead
  * This does not include arguments used by the base class' constructor (e.g. `interface`, `address`, `creator`, `chainId`, `db`), only the ones inherent to the contract itself
* The following arguments are tuples, one for each implemented contract function, that contain respectively:
  * The exact name of the function (`"getName"`)
  * A reference to the function itself (`&SimpleContract::getName`) - if you happen to have one or more overloads of the same function, you may need to specify which function is which using `static_cast` (e.g. `getNumber()` would be registered as `static_cast<uint256_t(SimpleContract::*)() const>(&SimpleContract::getNumber)`, while `getNumber(const uint256_t&)` would be registered as `static_cast<uint256_t(SimpleContract::*)(const uint256_t&) const>(&SimpleContract::getNumber)`)
  * The [state mutability](https://docs.soliditylang.org/en/latest/contracts.html#state-mutability) of said function, accessed by a `FunctionTypes` enum - available values are `"View"`, `"NonPayable"` and `"Payable"`
  * A string vector that is the list of arguments that the function takes, if any (or an empty list if none)

Every contract MUST have both `ConstructorArguments` AND `registerContract()` implemented in order to be registered, even if `ConstructorArguments` is empty (has no arguments at all).

## Registering contract events

If your contract has events, you should also register them so you can generate their ABI later on. Events are registered the same way as the contract class and its functions, but using a separate function from `ContractReflectionInterface` called `registerContractEvents()`:

```cpp
class SimpleContract : public DynamicContract {
  public:
    // ...
    static void registerContract() {
      static std::once_flag once;
      std::call_once(once, []() {
        // DynamicContract::registerContractMethods() called here
        ContractReflectionInterface::registerContractEvents<SimpleContract>(
          std::make_tuple("nameChanged", false, &SimpleContract::nameChanged, std::vector<std::string>{"name"}),
          std::make_tuple("numberChanged", false, &SimpleContract::numberChanged, std::vector<std::string>{"number"}),
          std::make_tuple("tupleChanged", false, &SimpleContract::tupleChanged, std::vector<std::string>{"tuple"})
        );
      });
    }
}
```

Inside the chevrons (`registerContractEvents<...>()`), like before, the only argument needed is the contract's class name (in this case, `SimpleContract` - again, *no quotes ""*).

Inside the parentheses (`registerContractEvents<>(...)`), much like the other register function, there is one tuple per implemented event, that contain respectively:

* The event's name (`"nameChanged"`)
* A flag indicating whether the event is anonymous or not
* A reference to the event itself (`&SimpleContract::nameChanged`)
* A string vector that is the list of parameters that the event takes, if any (or an empty list if none)

## Registering the contract in the blockchain

Finally, we go to `src/contract/customcontracts.h`, include our contract's header and add it to the `ContractTypes` tuple so it can be compiled alongside the blockchain:

```cpp
// ...some includes ...
#include "templates/simplecontract.h" // <--- Add this line

using ContractTypes = std::tuple<..., SimpleContract>; // <--- Add your contract here
```

