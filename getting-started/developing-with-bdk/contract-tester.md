---
description: A web application for testing your contracts
---

# Contract Tester

We have also developed a web application called [Contract Tester](https://github.com/AppLayerLabs/contract-tester), for testing the logic of customized contracts in a simple, fast and intuitive way.

It is assumed that you have read the previous sections before this one, have your environment setup and a local testnet already running in your machine. Some parts of the page may also require that you have read the Contract sections as it deals with deploying custom contracts.

## Cloning and deploying

First, clone the repository to your local device and move to it:

```bash
git clone https://github.com/AppLayerLabs/contract-tester
cd contract-tester
```

### Using Docker (recommended)

Similar to the BDK, you can deploy the web application in two ways: _using Docker_, or _manually_. We recommend using Docker, so make sure it is installed in your system:

* [Docker for Windows](https://docs.docker.com/docker-for-windows/install/)
* [Docker for Mac](https://docs.docker.com/docker-for-mac/install/)
* [Docker for Linux](https://docs.docker.com/desktop/install/linux-install/)

On Linux, you may need to run Docker commands as `sudo`, or you can follow [this post-install](https://docs.docker.com/engine/install/linux-postinstall/) so you don't have to. Command examples will be shown without `sudo` for simplicity purposes.

After installing Docker, build the image:

```bash
docker build -t contract-tester .
```

Then run the container:

```bash
docker run -p 3000:3000 contract-tester
```

#### VSCode + Docker extension

If you use VSCode as code editor, you can integrate it with the container. To do so, you need to install the [Docker extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker) and configure it to use the container. After installing it, there is a `docker-compose.yml` file on the root of the repository that you can use to build and run the container.

Right-click on it and select `Compose Up` (or run `docker compose up` in a terminal) to build and run the container, so the application will be deployed on `http://localhost:3000` (or the port you chose in the `docker-compose.yml` file). You can also use the `Compose Restart` option to restart the container, and `Compose Down` to stop and remove the container.

<figure><img src="../.gitbook/assets/VSCodeDockerExtension (2).gif" alt=""><figcaption><p>VSCode Docker integration</p></figcaption></figure>

### Manual setup

If you want to deploy the web application manually, you will need to install [NPM](https://www.npmjs.com/) along with a few dependencies:

```bash
sudo apt install npm
sudo npm install --global yarn
npm install next react@latest react-dom
```

Then, install the yarn dependencies and run the project:

```bash
yarn install
yarn build
yarn start
```

## Using the application

You should now be able to access the Contract Tester through `http://localhost:3000` (or the port you chose in the previous step). The application should look like this:

<figure><img src="../.gitbook/assets/TESTER_01.png" alt=""><figcaption><p>Main page of the application</p></figcaption></figure>

When you open the home page, it'll automatically ask for you to connect to MetaMask. You can also click on the "Connect MetaMask" button on the bottom of the left sidebar to do it manually.

The application has two main sections: **Contract Manager** and **Custom Contract**. The first one shows the function calls for the default `ContractManager` contract that we have supplied within the AppLayer project, and the second one is where you will call your custom contracts.

### Contract Manager

If you want to use the default contract, just type the required parameters for the function you want to call, and call it by clicking the "Call" button beside it. For example, let's create a new contract with the following parameters, based on the `SimpleContract` template:

* `NAME`: "SimpleContract"
* `SYMBOL`: "APPL"
* `DECIMALS`: 10
* `SUPPLY`: 100

<figure><img src="../.gitbook/assets/TESTER_02.png" alt=""><figcaption><p>Calling a function from Contract Manager</p></figcaption></figure>

After that, you should wait for the transaction to be mined and the contract to be created. You can check the transaction status on MetaMask. After the transaction is mined, you can click on the "getDeployedContracts" button to see the address of the contract you just created:

<figure><img src="../.gitbook/assets/ContractTesterContractManager2 (1).png" alt=""><figcaption><p>Results of the previous call</p></figcaption></figure>

If you want to create a custom version of `ContractManager`, you will need to follow these steps:

1. Deploy the custom contract to the network (see the Contract sections for more info on how to do it)
2. Take the custom contract address and put it at the top, in the “Contract Manager Address” field
3. Add the custom contract's ABI by clicking the button next to the "Enter your own JSON ABI" label, choosing your JSON ABI file, and then clicking "Upload"
4. Finally, click the “Get Functions” button to show the custom contract's functions and input fields

**NOTE**: we do _not_ advise you remove the “getDeployedContracts” function on any custom contract implementations you create. We have a statically placed table at the end of the page for quick reference to all the contracts deployed on the network by the contract manager. Use this to quickly fetch the addresses of deployed contracts to be able to interact with them.

### Custom Contract

If you want to interact with your own custom contracts, just remember these steps:

1. Develop the contract and deploy it to the network
2. Get the contract address by creating a new contract instance on the Contract Manager tab and clicking on the "Get Deployed Contracts" button
3. Use a tool to generate the contract JSON ABI
4. Go to the Custom Contract tab and paste the contract address in the "Contract Address" field and upload the ABI JSON file
5. Click on the "Get Functions" button and interact with the contract

First, make sure the contract is defined and added to the Contract Manager so you can deploy an instance of the contract. After deploying, grab the address from the “Deployed Contracts” list, click on the “Custom Contract” tab and repeat steps 1-4 as you did earlier in the Contract Manager page. Every time you want to change contracts, deploy a new one, change the address and ABI using the inputs, and get the functions.

**NOTE**: If you are passing an array as input values, just separate them using commas and the application will make all the necessary changes.

Let's use the `SimpleContract` example because it is already built-in to BDK and registered in `ContractManager`. First, develop the contract and deploy it to the network. After that, you can use the Contract Tester to interact with it. The Solidity code for the contract is:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8;

contract SimpleContract {
  address owner;
  string name;
  uint256 value;

  event nameChanged(string indexed name);
  event valueChanged(uint256 indexed value);

  constructor(string memory argName, uint256 argValue) {
    owner = msg.sender;
    name = argName;
    value = argValue;
  }

  function getName() public view returns(string memory) {
    return name;
  }

  function getValue() public view returns(uint256) {
    return value;
  }

  function setName(string memory argName) public {
    require(msg.sender == owner, "Not owner");
    name = argName;
  }

  function setValue(uint256 argValue) public {
    require(msg.sender == owner, "Not owner");
    value = argValue;
  }
}
```

We can use our own ABI generator tool (see the Contract sections), or a third-party one like [Remix IDE](https://remix.ethereum.org) to generate the contract ABI. If using Remix, first create a Solidity file (.sol extension) and paste the code above. After that, you can compile the contract by clicking on the "Compile" button on the left sidebar. You should see something like this:

<figure><img src="../.gitbook/assets/ContractTesterRemixIDE.png" alt=""><figcaption><p>Ethereum Remix IDE</p></figcaption></figure>

Now, you can click on the "ABI" button to copy the ABI JSON structure of the `SimpleContract`. Finally, you can just save it in a JSON file and use it in the Contract Tester. This is the JSON ABI for the `SimpleContract`:

```json
[
  {
    "inputs": [
    {
      "internalType": "string",
        "name": "argName",
        "type": "string"
    },
    {
      "internalType": "uint256",
      "name": "argValue",
      "type": "uint256"
    }
    ],
    "stateMutability": "nonpayable",
    "type": "constructor"
  },
  {
    "inputs": [],
    "name": "getName",
    "outputs": [
    {
      "internalType": "string",
      "name": "",
      "type": "string"
    }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "getValue",
    "outputs": [
    {
      "internalType": "uint256",
      "name": "",
      "type": "uint256"
    }
    ],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [
    {
      "internalType": "string",
      "name": "argName",
      "type": "string"
    }
    ],
    "name": "setName",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [
    {
      "internalType": "uint256",
      "name": "argValue",
      "type": "uint256"
    }
    ],
    "name": "setValue",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "anonymous": false,
    "inputs": [
    {
      "indexed": true,
      "internalType": "string",
      "name": "name",
      "type": "string"
    }
    ],
    "name": "nameChanged",
    "type": "event"
  },
  {
    "anonymous": false,
    "inputs": [
    {
      "indexed": true,
      "internalType": "uint256",
      "name": "value",
      "type": "uint256"
    }
    ],
    "name": "valueChanged",
    "type": "event"
  }
]
```

Now, you can use the Contract Tester to interact with the deployed `SimpleContract`. First, you need to get the address of the deployed contract. You can do that by going to the Contract Manager tab and clicking on the "Get Deployed Contracts" button. Remember that we already did this step and our contract address is `0x5b41CEf7F46A4a147e31150c3c5fFD077e54d0e1`. Then, go to the Custom Contract tab and paste the address in the "Contract Address" field. After that, you can upload the ABI JSON file and click on the "Get Functions" button. You should see something like this:

<figure><img src="../.gitbook/assets/TESTER_03.png" alt=""><figcaption><p>Testing calls to a custom contract</p></figcaption></figure>

Now you can interact with the contract. You can set the name and value by using the "setName" and "setValue" functions, and get the name and value by using the "getName" and "getValue" functions. You can also check the transactions on MetaMask.
