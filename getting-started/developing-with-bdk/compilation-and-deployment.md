---
description: Putting those bytes to work
---

# Compilation and deployment

After ensuring your environment is properly set up, you can now compile and deploy your own local testnet. This is strongly recommended, as it will ensure your network and contracts compile and work as they should before deploying your project on a more serious environment.

## Compiling

The following commands will build the project out of tree within a folder called `build_local_testnet`, which will be used later by the deploying script included in the project (see "Manual deploy" below - this is not entirely necessary as the script will rebuild the entire project if needed, but we recommend doing it this way to avoid having two separate builds taking up space).

First, create and move into the build folder:

```bash
mkdir build_local_testnet && cd build_local_testnet
```

Then, configure CMake to build inside the folder (the `..` points to the root folder's `CMakeLists.txt`):

```bash
cmake -DDEBUG=ON -DBUILD_TESTS=ON ..
```

The `DEBUG` flag enables debug symbols and the compiler's address sanitizer. It is `ON` by default. For release builds, this should be set to `OFF` as debug builds take a significant hit on performance. You can also set the `BUILD_TESTS` flag (`ON` by default) to toggle the compilation of unit tests, which can help with compilation times and/or RAM usage if not needed. Check the `CMakeLists.txt` file for more flags.

Finally, build the project. Adjust `-j$(nproc)` accordingly to your system's CPU cores and/or memory limits if necessary, as some parts of the project can get really heavy RAM-wise during compilation:

```bash
cmake --build . -- -j$(nproc)
```

## Running unit tests

After building, assuming `BUILD_TESTS` is `ON`, you can optionally run a test bench with the following command: `./src/bins/bdkd-tests/bdkd-tests -d yes` (the `-d yes` parameter will give a verbose output).

You can also use filter tags to test specific parts of the project (e.g. `./src/bins/bdkd-tests/bdkd-tests -d yes [utils]` will test all the components inside the `src/utils` folder, `[utils][tx]` will test only the transaction-related components inside that same folder, etc.). You can check all the available tags by doing a `grep -rw "\"\[.*\]\""` in the `tests` subfolder. You can also read the "BDK implementation" section for more details on how the project is structured.

## Documentation

The project also has built-in local documentation powered by [Doxygen](https://doxygen.nl). Make sure you have it installed in your system and run `doxygen` in the project's root folder to generate it. You can find the generated docs in the `docs` folder and open it in your browser of choice.

## Deploying

There are two ways to deploy an AppLayer node: *dockerized* and *manual*. Go back to the project's root folder and check the `scripts` subfolder - there are two main scripts there used for deploying the node. You can pick whichever one you prefer, depending on your needs.

### Dockerized deploy

You can deploy a node with Docker by running `./scripts/auto.sh`. Make sure you have both `docker` and `docker-compose` installed, as the script requires both to work. The script itself accepts several parameters. Running `./scripts/auto.sh help` will give you more info on each parameter.

### Manual deploy

To manually deploy a node, run `./scripts/AIO-setup.sh`. Make sure `tmux` is installed, as the script needs it to work. The script will create two folders at the project's root - `build_local_testnet` and `local_testnet`, used respectively for building and deploying a fresh new instance of a local testnet.

Running the script again will stop the testnet, rebuild it, update the binaries and restart it on the spot. If you wish to manually stop the testnet for some reason, run `tmux kill-server`. You can also read the script to find out the specific names of the tmux sessions to manually restart or stop accordingly.

**NOTE**: when re-deploying the testnet, if your wallet or RPC client keeps track of account nonce data, you must reset it as a network reset would set back their nonces to 0. [Here's how to do it in MetaMask, for example](https://support.metamask.io/hc/en-us/articles/360015488891-How-to-clear-your-account-activity-reset-account).

You can use the following flags when calling the manual script to customize deployment:

| Flag            | Description                                      | Default Value     |
| --------------- | ------------------------------------------------ | ----------------- |
| --clean         | Clean the build folder before building           | false             |
| --no-deploy     | Only build the project, don't deploy the network | false             |
| --debug=\<bool> | Build in debug mode                              | true              |
| --cores=\<int>  | Number of cores to use for building              | Maximum available |

As an example, `./scripts/AIO-setup.sh --clean --no-deploy --debug=false --cores=4` will clean the build folder, only build the project, build in release mode and use 4 cores for building. Remember that GCC uses around 1.5-2GB of RAM per core, so we recommend adjusting the number of cores according to the available RAM on your system for more stability.