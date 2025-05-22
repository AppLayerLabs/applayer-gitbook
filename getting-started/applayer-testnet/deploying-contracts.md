---
description: A hands-on guide for interacting with contracts in the AppLayer Testnet
---

# Deploying contracts (Testnet)

Here's a simple guide on how to deploy both precompiled (C++) and EVM (Solidity) contracts on the AppLayer Testnet. Check the Contracts section for deeper details on how it all works.

## Deploying C++ contracts

To deploy a C++ contract on the testnet, open Remix IDE and then compile this Solidity interface in it:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

interface ContractManager {
  struct Contract {
    string name;
    address addr;
  }

  function getDeployedContracts() external view returns(Contract[] memory);
  function getDeployedContractsForCreator(address creator) external view returns (Contract[] memory);
  function createNewERC20Contract(string calldata name, string calldata ticket, uint8 decimals, uint256 mintValue) external returns(address);
  function createNewNativeWrapperContract(string calldata erc20name, string calldata erc20ticker, uint8 erc20decimals) external returns(address);
  function createNewDEXV2PairContract() external returns(address);
  function createNewDEXV2FactoryContract(address feeToSetter) external returns(address);
  function createNewDEXV2Router02Contract(address factory, address nativeWrapper) external returns(address);
  function createNewERC721Contract(string calldata erc721name, string calldata erc721symbol) external returns(address);
}
```

This will allow you to call the `ContractManager` precompiled contract and use it to deploy any of the available precompiled contracts on the blockchain. Precompiled contract deploys cost 100,000 gas each.

To find out the address of your deployed contract, call the `getDeployedContractsForCreator()` function, passing the `ContractManager` address itself as the argument. The address for ContractManager is hardcoded to `0x0001cb47ea6d8b55fe44fdd6b1bdb579efb43e61`.

See the example video that deploys an ERC20 contract:

{% embed url="https://drive.google.com/file/d/1zYKqBOCqS_CoL2HM-jV1BzVWuirP2NkO/view?usp=drive_link" %}

## Deploying EVM contracts

Deploying a Solidity/EVM contract on the testnet is done just like Ethereum. **The AppLayer EVM is set to "Shanghai"**, so be sure to set Remix to the same version before compiling. Check the video below for an example on the following contract:

```solidity
// contracts/GLDToken.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract AnotherTestToken is ERC20 {
  constructor(uint256 initialSupply) ERC20("AnotherTestToken", "ATT") {
    _mint(msg.sender, initialSupply);
  }
}
```

{% embed url="https://drive.google.com/file/d/1Q1yV8J37fhn8CTfBlgYYCOXMz3FhHO3o/view?usp=drive_link" %}

## (Optional) Using randomness in EVM contracts

You can also use one of our on-chain precompiles called `BDKPrecompile` to fetch random numbers generated on the fly. It is only accessed by the EVM through the following Solidity interface:

```solidity
interface BDKPrecompile {
  function getRandom() external view returns (uint256);
}
```

The precompile is located at the address `0x1000000000000000000000000000100000000001`. To use the interface in your own contracts, you MUST specify this exact address in your contract's code when accessing it, like this:

```solidity
contract MyContract {
  function myFunction() public {
    // ...
    uint256 myRandomNumber = BDKPrecompile(0x1000000000000000000000000000100000000001).getRandom();
    // ...
  }
}
```

As an example, have a look at this Solidity code that represents one of our template contracts used for testing, called `RandomnessTest`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.17;

interface BDKPrecompile {
  function getRandom() external view returns (uint256);
}

contract RandomnessTest {
  uint256 private randomValue_;
  function setRandom() external {
    randomValue_ = BDKPrecompile(0x1000000000000000000000000000100000000001).getRandom();
  }
  function getRandom() view external returns (uint256) {
    return randomValue_;
  }
}
```

Compile this code in Remix IDE (**remember to set the EVM to "Shanghai"**, like in the previous step) and deploy the `RandomnessTest` contract. Once it is deployed call the `setRandom` function to initialize it, and then call the `getRandom` function to get a random number.

**If you are using the interface in a view function, be aware that two different executions will always result in a different value (if called by RPC).** This is done on purpose, as the random value is only decided when the transaction is included in a block and it is generated in a cryptographically secure manner, making it impossible to predict the value. For the `RandomnessTest` contract specifically, due to how it is coded, if you want a new random number you must call `setRandom` again before calling `getRandom` the next time.

See the following video as an example:

{% embed url="https://drive.google.com/file/d/1m1d7P2ibQTogSnq_72JTZjDvHat4Tnpd/view?usp=drive_link" %}
