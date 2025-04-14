---
description: How to deploy a contract with BDK and write tests for it
---

# Deploying and testing

Now that the contract is finished and implemented, all that's left to do is deploy it. Though we would also suggest generating the ABI of your contracts and writing some tests to make sure they're working as intended. This subchapter will explain how to do both.

## Generating the ABI

We have a tool that is compiled with the project that generates the ABI of your contract. Its executable is compiled from the `src/bins/contractabigenerator/main.cpp` file, with the following code snippet by default:

```cpp
#include "utils/jsonabi.h"

int main() { return JsonAbi::writeContractsToJson<ContractTypes>(); }
```

Compile the project and run `./contractabigenerator`. All of your Dynamic Contracts registered in `src/contract/customcontracts.h` (inside the `ContractTypes` tuple shown earlier) will have their ABIs generated inside the `ABI` folder, as `<ContractName>.json`.

If you want a finer control of which ABIs you want to generate, replace `ContractTypes` with the exact names of the contract classes that should be generated (note the `ContractManager` contract will *always* have its ABI generated regardless, as it is a vital part of the project):

```cpp
#include "utils/jsonabi.h"

int main() {
  return JsonAbi::writeContractsToJson<ERC20, ERC20Wrapper, ..., SimpleContract>();
}
```

## Testing the contract

We use the [catch2](https://github.com/catchorg/Catch2) framework to test our project as a whole, so it is expected to be a bit familiar with how it works (it's pretty easy!).

For testing contracts specifically, we have developed our own test suite called `SDKTestSuite`, in `tests/sdktestsuite.hpp` (the `.cpp` file is for testing the test suite itself). Using this test suite is highly recommended, as it simulates the environment of the blockchain (e.g. creating blocks, advancing the chain, taking care of the state and its variables, etc.) to make sure contracts are behaving the way they would in the actual deployed blockchain. Check the [Doxygen](https://doxygen.nl) docs, as well as the contract test files in the `tests/contract` subfolder, for more information on how to use the test suite.

First, create a new file in the `tests/contract` folder with the name of your contract - in our case, `tests/contract/simplecontract.cpp` (which already exists as it is part of the project).

Then, add the file to the `tests/CMakeLists.txt` file:

```cmake
set(TESTS_SOURCES
  # ...
  ${CMAKE_SOURCE_DIR}/tests/contract/simplecontract.cpp
  # ...
)
```

Finally, write your tests. In this case, for `SimpleContract`, tests should look like this:

```cpp
#include "../../src/libs/catch2/catch_amalgamated.hpp"

#include "../../src/contract/templates/simplecontract.h"

#include "../sdktestsuite.hpp"

namespace TSimpleContract {
  TEST_CASE("SimpleContract class", "[contract][simplecontract]") {
    SECTION("SimpleContract creation") {
      SDKTestSuite sdk = SDKTestSuite::createNewEnvironment("testSimpleContractCreation");
      Address simpleContract = sdk.deployContract<SimpleContract>(
        std::string("TestName"), uint256_t(19283187581),
        std::make_tuple(std::string("TupleName"), uint256_t(987654321))
      );
      std::string name = sdk.callViewFunction(simpleContract, &SimpleContract::getName);
      uint256_t number = sdk.callViewFunction(simpleContract, &SimpleContract::getNumber);
      std::tuple<std::string, uint256_t> tuple = sdk.callViewFunction(simpleContract, &SimpleContract::getTuple);
      REQUIRE(name == "TestName");
      REQUIRE(number == 19283187581);
      REQUIRE(std::get<0>(tuple) == "TupleName");
      REQUIRE(std::get<1>(tuple) == 987654321);
    }

    SECTION("SimpleContract setName, setNumber and setTuple") {
      SDKTestSuite sdk = SDKTestSuite::createNewEnvironment("testSimpleContractSetNameNumberAndTuple");
      Address simpleContract = sdk.deployContract<SimpleContract>(
        std::string("TestName"), uint256_t(19283187581),
        std::make_tuple(std::string("TupleName"), uint256_t(987654321))
      );

      std::string name = sdk.callViewFunction(simpleContract, &SimpleContract::getName);
      uint256_t number = sdk.callViewFunction(simpleContract, &SimpleContract::getNumber);
      std::tuple<std::string, uint256_t> tuple = sdk.callViewFunction(simpleContract, &SimpleContract::getTuple);
      REQUIRE(name == "TestName");
      REQUIRE(number == 19283187581);
      REQUIRE(std::get<0>(tuple) == "TupleName");
      REQUIRE(std::get<1>(tuple) == 987654321);

      Hash nameTx = sdk.callFunction(simpleContract, &SimpleContract::setName, std::string("TryThisName"));
      Hash numberTx = sdk.callFunction(simpleContract, &SimpleContract::setNumber, uint256_t("918258172319061203818967178162134821351"));
      Hash tupleTx = sdk.callFunction(simpleContract, &SimpleContract::setTuple, std::make_tuple(std::string("AnotherName"), uint256_t(999999999)));
      name = sdk.callViewFunction(simpleContract, &SimpleContract::getName);
      number = sdk.callViewFunction(simpleContract, &SimpleContract::getNumber);
      tuple = sdk.callViewFunction(simpleContract, &SimpleContract::getTuple);
      REQUIRE(name == "TryThisName");
      REQUIRE(number == uint256_t("918258172319061203818967178162134821351"));
      REQUIRE(std::get<0>(tuple) == "AnotherName");
      REQUIRE(std::get<1>(tuple) == 999999999);

      auto nameEvent = sdk.getEventsEmittedByTx(nameTx, &SimpleContract::nameChanged,
        std::make_tuple(EventParam<std::string, true>("TryThisName"))
      );
      auto numberEvent = sdk.getEventsEmittedByTx(numberTx, &SimpleContract::numberChanged,
        std::make_tuple(EventParam<uint256_t, false>(uint256_t("918258172319061203818967178162134821351")))
      );
      auto tupleEvent = sdk.getEventsEmittedByTx(tupleTx, &SimpleContract::tupleChanged,
        std::make_tuple(EventParam<std::tuple<std::string, uint256_t>, true>(std::make_tuple("AnotherName", uint256_t(999999999))))
      );
      REQUIRE(nameEvent.size() == 1);
      REQUIRE(numberEvent.size() == 1);
      REQUIRE(tupleEvent.size() == 1);
    }
  }
}
```

Keep in mind that we're not accessing the contract directly, we're interacting with it through `SDKTestSuite`, and that requires us to parse its inputs and outputs accordingly.

In order to run your tests, compile the project as you normally would, and then run `./src/bins/bdkd-tests/bdkd-tests -d yes [simplecontract]` from within your build directory. The `[simplecontract]` tag forces only these specific tests for `SimpleContract` to run (this is set in the `TEST_CASE()` lines in the example above). The `-d yes` flag makes it more verbose, showing exactly which test case is being run at the moment.

As a bonus, even though this whole chapter is focused on precompiled contracts, the `SDKTestSuite` also has a function that can deploy a contract's bytecode directly, called `SDKTestSuite::deployBytecode()`, which takes a `Bytes` object as a parameter that represents the contract's bytecode.

## Deploying the blockchain

For real-world testing, go back to the project's root and deploy the blockchain. See "Developing with BDK" for more information. Your contract should be initialized alongside the blockchain, ready to be interacted with.
