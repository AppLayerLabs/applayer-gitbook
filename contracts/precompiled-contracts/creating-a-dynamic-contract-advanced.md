---
description: A walkthrough on a more advanced DynamicContract usage
---

# Creating a Dynamic Contract (Advanced)

Let's create another Dynamic Contract that can be used for depositing and withdrawing ERC20 tokens. This subchapter assumes you went through the previous (Simple) one first, as most of the heavy explanations are there.

This example uses an already existing contract within the project - the `ERC20Wrapper` contract. This is due to the fact that `ERC20Wrapper` is a very simple contract, while it shows differences between Solidity and AppLayer contracts when calling other contracts. For reference, check the `erc20wrapper.h` and `erc20wrapper.cpp` files in `src/contract/templates`.

## Solidity Example

We'll be using the following Solidity code as a reference:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract ERC20Wrapper {
  mapping (address => mapping (address => uint256)) private _tokensAndBalances;

  function getContractBalance(address token) public view returns(uint256) {
    IERC20 erc20 = IERC20(token);
    return erc20.balanceOf(address(this));
  }

  function getUserBalance(address token, address user) public view returns (uint256) {
    return _tokensAndBalances[token][user];
  }

  function withdraw(address token, uint256 value) public {
    require(_tokensAndBalances[token][msg.sender] >= value, "User doesn't have enough balance");
    IERC20 erc20 = IERC20(token);
    _tokensAndBalances[token][msg.sender] -= value;
    erc20.transfer(msg.sender, value);
  }

  function transferTo(address token, address to, uint256 value) public {
    require(_tokensAndBalances[token][msg.sender] >= value, "User doesn't have enough balance");
    IERC20 erc20 = IERC20(token);
    _tokensAndBalances[token][msg.sender] -= value;
    erc20.transfer(to, value);
  }

  function deposit(address token, uint256 value) public {
    IERC20 erc20 = IERC20(token);
    erc20.transferFrom(msg.sender, address(this), value);
    _tokensAndBalances[token][msg.sender] += value;
  }
}
```

## Creating the files

Create the header and source files (`erc20wrapper.h` and `erc20wrapper.cpp`, as stated above) and add them to `CMakeLists.txt`:

```cmake
set(CONTRACT_HEADERS
  # ...
  ${CMAKE_SOURCE_DIR}/src/contract/templates/erc20wrapper.h
  # ...
)
set(CONTRACT_SOURCES
  # ...
  ${CMAKE_SOURCE_DIR}/src/contract/templates/erc20wrapper.cpp
  # ...
)
```

## Creating the contract header and registering

Inside `erc20wrapper.h`, let's implement the header (comments were taken out so it's easier to read):

```cpp
#ifndef ERC20WRAPPER_H
#define ERC20WRAPPER_H

#include <memory>
#include <tuple>

#include "../../utils/db.h"
#include "../../utils/utils.h"
#include "../abi.h"
#include "../contractmanager.h"
#include "../dynamiccontract.h"
#include "../variables/safeunorderedmap.h"
#include "standards/erc20.h"

class ERC20Wrapper : public DynamicContract {
  private:
    SafeUnorderedMap<Address, boost::unordered_flat_map<Address, uint256_t, SafeHash>> tokensAndBalances_;
    void registerContractFunctions() override;

  public:
    using ConstructorArguments = std::tuple<>;

    ERC20Wrapper(const Address& contractAddress, const DB& db);
    ERC20Wrapper(const Address& address, const Address& creator, const uint64_t& chainId);
    ~ERC20Wrapper() override;

    static void registerContract() {
      static std::once_flag once;
      std::call_once(once, []() {
        DynamicContract::registerContractMethods<ERC20Wrapper>(
          std::vector<std::string>{},
          std::make_tuple("getContractBalance", &ERC20Wrapper::getContractBalance, FunctionTypes::View, std::vector<std::string>{"token"}),
          std::make_tuple("getUserBalance", &ERC20Wrapper::getUserBalance, FunctionTypes::View, std::vector<std::string>{"token", "user"}),
          std::make_tuple("withdraw", &ERC20Wrapper::withdraw, FunctionTypes::NonPayable, std::vector<std::string>{"token", "value"}),
          std::make_tuple("transferTo", &ERC20Wrapper::transferTo, FunctionTypes::NonPayable, std::vector<std::string>{"token", "to", "value"}),
          std::make_tuple("deposit", &ERC20Wrapper::deposit, FunctionTypes::NonPayable, std::vector<std::string>{"token", "value"})
        );
      });
    }

    uint256_t getContractBalance(const Address& token) const;
    uint256_t getUserBalance(const Address& token, const Address& user) const;
    void withdraw(const Address& token, const uint256_t& value);
    void transferTo(const Address& token, const Address& to, const uint256_t& value);
    void deposit(const Address& token, const uint256_t& value);

    DBBatch dump() const override;
};

#endif
```

Here, we recreated the contract's functions but also added a few extra functions (explained in the previous sections). In short, we create:

* Two constructors - one for creating the contract from scratch, and another for loading it from the database
* The `ConstructorArguments` tuple, `registerContract()` and `registerContractFunctions()` functions for proper contract registering (notice that the tuple is required, even though it's empty)
* The `dump()` function for saving the contract's variables in the database
* Private SafeVariables (in this case, `SafeUnorderedMap`) to handle the contract's data
* The contract's functions according to the Solidity signatures

Like with `SimpleContract`, you must include your contract's header in `src/contract/customcontracts.h` to register it, and check it's set to generate its ABI in `src/bins/contractabigenerator/main.cpp`. In this specific case for `ERC20Wrapper`, it's assumed that both steps are already done, but it's good to check again just in case.

## Implementing contract constructors and dump function

Inside `erc20wrapper.cpp`, let's implement both constructors and the dumping function:

```cpp
#include "erc20wrapper.h"

ERC20Wrapper::ERC20Wrapper(const Address& contractAddress, const DB& db
) : DynamicContract(contractAddress, db), tokensAndBalances_(this)
{
  for (const auto& dbEntry : db.getBatch(this->getNewPrefix("tokensAndBalances_"))) {
    View<Bytes> valueView(dbEntry.value);
    this->tokensAndBalances_[Address(dbEntry.key)][Address(valueView.subspan(0, 20))] = Utils::fromBigEndian<uint256_t>(valueView.subspan(20));
  }

  this->tokensAndBalances_.commit();

  registerContractFunctions();

  this->tokensAndBalances_.enableRegister();
}

ERC20Wrapper::ERC20Wrapper(const Address& address, const Address& creator, const uint64_t& chainId
) : DynamicContract("ERC20Wrapper", address, creator, chainId), tokensAndBalances_(this)
{
  this->tokensAndBalances_.commit();

  registerContractFunctions();

  this->tokensAndBalances_.enableRegister();
}

ERC20Wrapper::~ERC20Wrapper() {}

DBBatch ERC20Wrapper::dump() const {
  DBBatch dbBatch = BaseContract::dump();
  for (auto i = tokensAndBalances_.cbegin(); i != tokensAndBalances_.cend(); ++i) {
    for (auto j = i->second.cbegin(); j != i->second.cend(); ++j) {
      const auto& key = i->first;
      Bytes value = j->first.asBytes();
      Utils::appendBytes(value, Utils::uintToBytes(j->second));
      dbBatch.push_back(key, value, this->getNewPrefix("tokensAndBalances_"));
    }
  }
  return dbBatch;
}
```

One constructor will create a new contract from scratch, as there is no previous existing contract to load, while the other will load the contract from the database when it already exists there. On both cases you are required to initialize, commit and enable registering for *all* the existing SafeVariables in your contract's constructors, as well as calling `registerContractFunctions()`, all in the same order as explained in the previous subchapter. The dumping function on the other hand is responsible for saving the current information within the contract back to the database.

Notice that your contract's name ("ERC20Wrapper") is the same as your contract's class name (`ERC20Wrapper`) - again, just like with `SimpleContract`, this match is **mandatory**, otherwise a segfault will happen. `getNewPrefix()` does the same as `getDBPrefix()`, but with a user-defined string appended to it, so this would be equivalent to "`DBPrefix::contracts` + the contract's address + `tokensAndBalances_`" (as a raw bytes string).

## Implementing contract functions

This step is pretty straightforward, we just follow the rules explained previously:

```cpp
uint256_t ERC20Wrapper::getContractBalance(const Address& token) const {
  return this->callContractViewFunction(token, &ERC20::balanceOf, this->getContractAddress());
}

uint256_t ERC20Wrapper::getUserBalance(const Address& token, const Address& user) const {
  auto it = this->tokensAndBalances_.find(token);
  if (it == this->tokensAndBalances_.cend()) return 0;
  auto itUser = it->second.find(user);
  if (itUser == it->second.cend()) return 0;
  return itUser->second;
}

void ERC20Wrapper::withdraw(const Address& token, const uint256_t& value) {
  auto it = this->tokensAndBalances_.find(token);
  if (it == this->tokensAndBalances_.end()) throw DynamicException("Token not found");
  auto itUser = it->second.find(this->getCaller());
  if (itUser == it->second.end()) throw DynamicException("User not found");
  if (itUser->second <= value) throw DynamicException("ERC20Wrapper: Not enough balance");
  itUser->second -= value;
  this->callContractFunction(token, &ERC20::transfer, this->getCaller(), value);
}

void ERC20Wrapper::transferTo(const Address& token, const Address& to, const uint256_t& value) {
  auto it = this->tokensAndBalances_.find(token);
  if (it == this->tokensAndBalances_.end()) throw DynamicException("Token not found");
  auto itUser = it->second.find(this->getCaller());
  if (itUser == it->second.end()) throw DynamicException("User not found");
  if (itUser->second <= value) throw DynamicException("ERC20Wrapper: Not enough balance");
  itUser->second -= value;
  this->callContractFunction(token, &ERC20::transfer, to, value);
}

void ERC20Wrapper::deposit(const Address& token, const uint256_t& value) {
  this->callContractFunction(token, &ERC20::transferFrom, this->getCaller(), this->getContractAddress(), value);
  this->tokensAndBalances_[token][this->getCaller()] += value;
}
```

### Calling functions from another contract

Note that, in the example above, some functions are calling functions from another contract. This is done by calling `callContractViewFunction()` (**for view functions**) and `callContractFunction()`(**for non-view/callable functions**), both of which require the following arguments:

* The other contract's address (in this case, `token`)
* A reference to the function that will be called - in this case:
  * `getContractBalance()` calls `ERC20::balanceOf()`
  * `withdraw()` and `transferTo()` call `ERC20::transfer()`
  * `deposit()` calls`ERC20::transferFrom()`
* The function's arguments, if there are any - in this case:
  * `ERC20::balanceOf()` will receive our contract's own address as `getContractAddress()`
  * `ERC20::transfer()` will receive the receiver address as `to` or `getCaller()`, and the value to be transferred as `value`
  * `ERC20::transferFrom()` will receive the sender's address as `getCaller()`, the receiver's address as `getContractAddress()`, and the value to be transferred as `value`

Alternatively, *for view functions specifically*, you can get a pointer to the other contract with `getContract()` and call the function directly from it, passing along any required arguments. The `getContract()` function itself is automatically protected in the case of casting a wrong typed contract or calling an inexistent contract. So the implementation of `getContractBalance()` could also be done like this:

```cpp
uint256_t ERC20Wrapper::getContractBalance(const Address& token) const {
  auto* ERC20Token = this->getContract<ERC20>(token);
  return ERC20Token->balanceOf(this->getContractAddress());
}
```

As this contract is consuming the ERC20 balance of another contract, you first need to approve the contract to spend the tokens. This can be done in the same manner as Solidity.

### Creating contracts on the fly

As a bonus, it's possible for a Dynamic Contract to create another Dynamic Contract, with the `callCreateContract()` function. That way you can have, for example, a function that creates another contract on the fly and retrieve its address (see below), but this is also useful for more advanced things like contract factories, where you can have a templated function that creates contracts on the fly based on the type and parameters passed to it.

The function should be used like this:

```cpp
Address contract = this->callCreateContract<ERC20>(this->getContractAddress(), 0, 0, 0, "TestToken", "TST", 18, 1000000000000000000);
```

This creates a new `ERC20` contract with the respective parameters:

* The caller address, in this case our own contract's address as `this->getContractAddress()`
* The gas value, in this case `0`
* The gas price value, also `0`
* The caller/transaction value, again, `0`
* The new contract's constructor parameters, in this case an `ERC20` contract needs the token name (`"TestToken"`), its ticker (`"TST"`), number of decimals (`18`), and the amount of tokens that will be minted at creation (`1000000000000000000`, which equals exactly 1 TST with 18 decimals)

## Registering contract functions

Once we're done with implementing the contract, we must register it. We've already coded the `ConstructorArguments` tuple and the `registerContract()` function in the header, so all that's left is to override `registerContractFunctions()` so we can register the contract's functions:

```cpp
void ERC20Wrapper::registerContractFunctions() {
  registerContract();
  this->registerMemberFunctions(
    std::make_tuple("getContractBalance", &ERC20Wrapper::getContractBalance, FunctionTypes::View, this),
    std::make_tuple("getUserBalance", &ERC20Wrapper::getUserBalance, FunctionTypes::View, this),
    std::make_tuple("withdraw", &ERC20Wrapper::withdraw, FunctionTypes::NonPayable, this),
    std::make_tuple("transferTo", &ERC20Wrapper::transferTo, FunctionTypes::NonPayable, this),
    std::make_tuple("deposit", &ERC20Wrapper::deposit, FunctionTypes::NonPayable, this)
  );
}
```

## Compiling and deploying

Finally, go back to the project's root, deploy your local network and your contract using the chain owner's private key, then test your contract's compatibility with your favorite frontend tool. If you wish, you can also write tests for your contract. Check the `tests` folder for more information.
