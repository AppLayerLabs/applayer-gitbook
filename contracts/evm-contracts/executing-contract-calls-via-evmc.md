---
description: How AppLayer's EVM contracts interact with the native blockchain state
---

# Executing contract calls via EVMC

To execute a contract call within AppLayer's EVM environment, we use the `evmc_execute()` function from the EVMC library. This function orchestrates the execution of contract bytecode and interfaces directly with the virtual machine:

```cpp
static inline struct evmc_result evmc_execute(
  struct evmc_vm* vm, const struct evmc_host_interface* host,
  struct evmc_host_context* context, enum evmc_revision rev,
  const struct evmc_message* msg, uint8_t const* code, size_t code_size
);
```

It's crucial to understand that the VM itself is *stateless* - it does not maintain any information about the contracts' states or their data. Its role is strictly to interpret and execute bytecode according to the Ethereum protocol specifications.

To enable the stateless VM to interact with the blockchain's state (such as querying storage keys, account balances, or initiating further contract calls), we must provide it with access to the state through the `evmc_host_interface` and `evmc_host_context` structs. The former contains a set of callback functions that the VM can use to query or modify the blockchain's state:

```cpp
struct evmc_host_interface {
  evmc_account_exists_fn account_exists;
  evmc_get_storage_fn get_storage;
  evmc_set_storage_fn set_storage;
  evmc_get_balance_fn get_balance;
  evmc_get_code_size_fn get_code_size;
  evmc_get_code_hash_fn get_code_hash;
  evmc_copy_code_fn copy_code;
  evmc_selfdestruct_fn selfdestruct;
  evmc_call_fn call;
  evmc_get_tx_context_fn get_tx_context;
  evmc_get_block_hash_fn get_block_hash;
  evmc_emit_log_fn emit_log;
  evmc_access_account_fn access_account;
  evmc_access_storage_fn access_storage;
  evmc_get_transient_storage_fn get_transient_storage;
  evmc_set_transient_storage_fn set_transient_storage;
};
```

Within AppLayer's BDK (more specifically inside `EvmExecutionContext::executeEvmcMessage()`), we use `evmc_execute()` like this for example:

```cpp
evmc::Result result (evmc_execute(this->vm_, &this->get_interface(), this->to_context(),
evmc_revision::EVMC_LATEST_STABLE_REVISION, &msg, recipientAcc.code.data(), recipientAcc.code.size()));
```

`ContractHost` is casted into `evmc_host_interface` to provide the VM with the necessary state access functions. This allows the VM to interact directly with the state and execute contract calls within AppLayer's native environment.
