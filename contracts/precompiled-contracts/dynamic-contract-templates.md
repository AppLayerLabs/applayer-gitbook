---
description: Which kinds of Dynamic Contract templates and precompiles are offered by AppLayer
---

# Dynamic Contract Templates and Precompiles

AppLayer's BDK provides ready-to-use templates for the following Dynamic Contracts:

* Under `src/contract/templates/standards`:
  * `ERC20` (template for an ERC20 token)
  * `ERC721` (template for an ERC721 token)
  * `ERC721URIStorage` (template for managing ERC721 token storage, converted directly from OpenZeppelin)
  * `IERC721Receiver` (template for an interface that enables safeTransfer support for ERC721 tokens, converted directly from OpenZeppelin)
* Under `src/contract/templates/dexv2`:
  * `DEXV2Factory` (template for a DEX factory)
  * `DEXV2Library` (namespace for commonly used DEX functions)
  * `DEXV2Pair` (template for a DEX contract pair)
  * `DEXV2Router02` (template for a DEX contract router)
  * `UQ112x112` (namespace for dealing with fixed point fractions in DEX contracts)
* Under `src/contract/templates`:
  * `ERC20Wrapper` (template for an ERC20 wrapper)
  * `MintableERC20` (template for a mintable ERC20 token)
  * `NativeWrapper` (template for a native asset wrapper)
  * `Ownable` (template for managing authorized access to certain contract calls, converted directly from OpenZeppelin)

We also have contract precompiles under the `src/contract/templates/precompiles` folder, inside the `precompiles` namespace. They are stateful in the C++ side (so you only have to include their headers) and stateless in the EVM side (so you have to call them through a specific address). Here's a list of the available precompiles and their respective addresses, as per the [official EVM Codes reference list](https://www.evm.codes/precompiled):

* `ecrecover` (0x01)
* `sha256` (0x02)
* `ripemd160` (0x03)
* `modexp` (0x05)
* `blake2f` (0x09)

There are also specific contracts that only exist for internal testing purposes and are not meant to be used as templates:

* `BuildTheVoid` and `BTV*` (contracts specific to the on-chain game Build The Void)
* `ERC721Test` (derivative contract meant to test the capabilities of the ERC721 template)
* `Pebble` (contract specific to the on-chain NFT minting game Pebble)
* `RandomnessTest` (contract for testing random number generation)
* `SimpleContract` (what it says on the tin - a simple contract, used for both testing and teaching purposes)
* `SnailTracer` / `SnailTracerOptimized` (C++ conversions of the [SnailTracer](https://github.com/karalabe/snailtracer) contract, used for benchmarking purposes)
* `TestThrowVars` (contract meant to test SafeVariable commit/revert functionality using exception throwing)
* `ThrowTest*` (contracts meant to test nested call revert functionality)
