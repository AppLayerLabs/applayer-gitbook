---
description: Prepping up the space
---

# Environment setup

Before compiling BDK from source, we must ensure all of its dependencies are installed. Most of those dependencies come from the operating system (as in "directly from a Linux distro's repos"), but some of them are external and compiled alongside the project itself.

## Forking

Head over to the [BDK repository on Github](https://github.com/AppLayerLabs/bdk-cpp) and click the "Fork" button. You now have your own copy of the project. After that, clone your forked repository to your local machine with `git clone https://github.com/YOUR_USER_NAME/bdk-cpp.git`. Now you're ready to start developing your own local blockchain.

<figure><img src="../.gitbook/assets/fork.png" alt=""><figcaption><p>Go on and fork it!</p></figcaption></figure>

## Setup

You can setup your local environment in two ways: _using Docker_, or _manually_. Manual setup has instructions for APT-based distros (e.g. Debian, Ubuntu, Mint, etc.), but other distros should work as long as you meet the minimum version requirements for all installed dependencies.

### Docker (recommended)

Using Docker is the recommended way to develop with the BDK. It will ensure that you have the correct environment to build and deploy the network, without worrying about dependencies or which host distro you're using.

First, install Docker on your system (if you don't have it installed already). Instructions vary depending on the operating system you're using:

* [Docker for Windows](https://docs.docker.com/docker-for-windows/install/)
* [Docker for Mac](https://docs.docker.com/docker-for-mac/install/)
* [Docker for Linux](https://docs.docker.com/desktop/install/linux-install/)

On Linux, you may need to run Docker commands as `sudo`, or you can follow [this post-install](https://docs.docker.com/engine/install/linux-postinstall/) so you don't have to. Command examples will be shown without `sudo` for simplicity purposes.

Once Docker is installed, go to the root directory of your cloned repository (where the `Dockerfile` is located), and run the following command:

```bash
docker build -t bdk-cpp-dev:latest .
```

This will build the image and tag it as `bdk-cpp-dev:latest`. You can change the tag to whatever you want, but remember to change it at the next step.

After building the image, run a container with the following command:

```bash
# For Linux/Mac
docker run -it --name bdk-cpp -v $(pwd):/bdk-volume -p 8080-8099:8080-8099 -p 8110-8111:8110-8111 bdk-cpp-dev:latest
# For Windows
docker run -it --name bdk-cpp -v %cd%:/bdk-volume -p 8080-8099:8080-8099 -p 8110-8111:8110-8111 bdk-cpp-dev:latest
```

where:

* `--name bdk-cpp` is an optional label for easier handling of the container (instead of using its ID directly)
* `$(pwd)` or `%cd%` is the absolute/full path to your repository's folder
* `:/bdk-volume` is the path inside the container where the BDK will be mounted. This volume is synced with the `bdk-cpp` folder inside the container
* The `-p` flags expose the ports used by the nodes. The example exposes the default ports 8080-8099 and 8110-8111 - if you use different ports, change them accordingly

When running the container, you will be logged in as `root` and will be able to develop, build and deploy the network within the container. Remember that we are using our local repo as a volume, so every change in the local folder will be reflected to the container in real time, and vice-versa (so you can develop outside and use the container only for build and deploy). You can also integrate the container with your favorite IDE or editor.

#### VSCode + Docker extension

To integrate the container with VSCode, you need to install the [Docker extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker) and configure it to use the container. After installing it, there is a `docker-compose.yml` file in the prject's root folder that you can use to build and run the container. The only thing that you need to do is to change the `volumes` section to point to your local SDK folder:

```yaml
volumes:
  - /path/to/your/sdk:/orbitersdk-volume
```

After editing the `docker-compose.yml` file, right-click on it and select `Compose Up` (or run `docker compose up` in a terminal) to build and run the container so you can start developing on it. Click on the Docker extension icon on the left side of the VSCode window and you will see the container running. You can also right-click on the container and select `Attach Shell` to open a terminal on the container.

<figure><img src="../.gitbook/assets/VSCodeDockerExtension (1).gif" alt=""><figcaption></figcaption></figure>

### Manual setup

If you don't want to use Docker for some reason, you can also build the project manually in your own system. Make sure your system provides at least the following dependencies:

#### Toolchain binaries

* **git**
* **GCC** with support for **C++23** or higher
* **Make**
* **CMake 3.19.0** or higher
* **Protobuf** (protoc + grpc_cpp_plugin)
* **tmux** (for deploying)
* (optional) **ninja** if you prefer it over make
* (optional) **mold** if you prefer it over ld
* (optional) **doxygen** for generating docs
* (optional) **clang-tidy** for linting

#### Libraries

* **Boost 1.83** or higher (components: *chrono, filesystem, program-options, system, thread, nowide*)
* **OpenSSL 1.1.1** / **libssl 1.1.1** or higher
* **libzstd**
* **CryptoPP 8.2.0** or higher
* **libscrypt**
 * **libc-ares**
* **gRPC** (libgrpc and libgrpc++)
* **secp256k1**
* **ethash** + **keccak**
* **EVMOne** + **EVMC**
* **Speedb**

Dependency versions should suffice out-of-the-box for at least the following distros (or greater, including their derivatives):

* **Debian 13 (Trixie)**
* **Ubuntu 24.04 LTS (Noble Numbat)**
* **Linux Mint 22 (Wilma)**
* **Fedora 40**
* Any rolling release distro from around **May 2024** onwards (check the repos to be sure)

#### The deps.sh script

We provide a script called `scripts/deps.sh` which automates the process of checking and installing the project's dependencies on the system. We strongly recommend using it if building manually, unless you know what you're doing.

The script accepts the following arguments (they should be run one at a time, e.g. `deps.sh --arg`):

* `--check`: scans the system and confirms which dependencies are properly installed
* `--install`: installs missing dependencies not found during check
* `--cleanext`: clean up external dependencies (those compiled alongside the project), in case rebuilding them from a clean slate is necessary

The script expects dependencies to be installed either on `/usr` or `/usr/local`, giving preference to the latter if it finds anything there. That way you can use a higher version of a dependency while still keeping your distro's default one.

**Please note that installing most dependencies through the script only works on APT-based distros** (Debian, Ubuntu and derivatives) - you can still check the dependencies on any distro and install the few ones labeled as "external" (those are fetched through `git`), but if you're on a distro with another package manager and/or a distro older than one of the minimum ones listed above, you're on your own.

#### Debian and GCC caveat

For Debian specifically, you can (and should) use `update-alternatives` to register and set your GCC version to a more up-to-date build if required.

If you're using a self-compiled GCC build out of the system path (e.g. `--prefix=/usr/local/gcc-X.Y.Z` instead of `--prefix=/usr/local`), don't forget to export its installation paths in your `PATH` and `LD_LIBRARY_PATH` env vars (to prevent e.g. `version GLIBCXX_.../CXXABI_... not found` errors). Put something like this in your `~/.bashrc` file for example, changing the version accordingly to whichever one you have installed:

```bash
# For GCC in /usr/local
export LD_LIBRARY_PATH=/usr/local/lib64:$LD_LIBRARY_PATH

# For self-contained GCC outside /usr/local
export PATH=/usr/local/gcc-14.2.0/bin:$PATH
export LD_LIBRARY_PATH=/usr/local/gcc-14.2.0/lib64:$LD_LIBRARY_PATH
```