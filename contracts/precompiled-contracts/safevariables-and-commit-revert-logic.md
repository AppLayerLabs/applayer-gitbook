---
description: How we simulate Solidity commit/revert logic in precompiled contracts
---

# SafeVariables and commit/revert logic

In C++, when you call a function that modifies a variable and then throws an exception, the changed variable is *not* reverted automatically. Consider the following example:

```cpp
MyClass::updateValueAndThrow(const uint64_t key, const uint64_t value) {
  // Suppose the original value of myMap[key] is 5
  this->myMap[key] = value; // e.g. 10
  throw std::runtime_error("Error");
  // myMap[key] remains 10, but it shouldn't because it threw
}
```

Even if you use a try/catch block, the value of `myMap[key]` will be permanently changed and won't roll back on its own, because that's how C++ works - it doesn't go out of its way to do things you didn't explicitly tell it to do. You would have to manually roll this logic by storing the old value in a temporary variable before changing it, and implementing specific logic to assign it back again in case of an error. The previous example could theoretically be coded like this:

```cpp
MyClass::updateValueAndThrow(const uint64_t key, const uint64_t value) {
  const uint64_t oldValue = this->myMap[key]; // Store the old value of myMap[key] first
  try {
    this->myMap[key] = value; // myMap[key] is now 10
    throw std::runtime_error("Error"); // Something went wrong
  } catch (std::exception& e) {
    this->myMap[key] = oldValue; // Assign the original value back, myMap[key] is now 5 again
  }
}
```

For Protocol Contracts, that's most of the required context. Dynamic Contracts, however, automatically provide this kind of functionality with the use of special types called **SafeVariables**, declared inside the `src/contract/variables` folder. Each Dynamic Contract includes a vector of references to SafeVariables, which is used to register used variables within a specific function call.

All SafeVariables inherit from the `SafeBase` class, which adhere to the following rules:

* Have two internal variables: one for the current value and another for the previous value
  * Optionally, if required, an undo stack for dealing with more complex variables such as containers
* Must override the `commit()` and `revert()` functions
  * `commit()` should keep the current value as-is and either discard the previous one or equal it to the current, depending on the implementation details
  * `revert()` should do the opposite - discard the current value and set it back to the previous one (also applying the undo stack if it has one, again, depending on the implementation details)
* Must be declared as *private* within contracts and initialized with `this` as the parameter - this enables the SafeVariable to mark itself as used in the `ContractManager`, allowing proper reversion of changes made to the contract
* Must call `markAsUsed()` on any and all non-`const` functions that it has (including ones that don't actually change the value), allowing the contract to properly register the variable as used within the function call, and properly revert or commit changes made to the contract accordingly
* When initialized with values in a contract's constructor, must call `commit()` followed by `enableRegister()` so the commit/revert functionality can actually work (see existing contracts for more info)

## Types of SafeVariables

We provide the following SafeVariable types, fully functional and ready for use in contracts:

* `SafeAddress` - abstracts a 20-byte address
* `SafeArray` - abstracts an array
* `SafeBool` - abstracts a boolean
* `SafeBytes` - abstracts a raw bytes string
* `SafeInt` and `SafeUint` - abstract a range of signed/unsigned integers (see Solidity ABI)
* `SafeString` - abstracts a string (either literal or raw bytes)
* `SafeTuple` - abstracts a tuple/struct of any type
* `SafeUnorderedMap` - abstracts a mapping
* `SafeVector` - abstracts a vector

### Caveats

Some specific non-`const` functions from certain SafeVars are NOT considered "safe" at the moment - this means commit/revert logic does NOT work properly on them due to implementation issues. We advise caution for now if you want to use them, preferably in a read-only manner (as in, do not alter the value itself if using one of those):

* `SafeUnorderedMap::find()`
* `SafeUnorderedMap::begin()`
* `SafeUnorderedMap::end()`
