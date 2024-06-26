## GGX Chain Systemd Installation Guide

If you are viewing this from the console, here is the [permanent link](https://github.com/ggxchain/documentation/blob/main/src/sydney-testnet/validator-guides/how-to-setup-a-validator-node.md) for better readability and convenience.

Repository: https://github.com/ggxchain/ggxnode

* _We're currently on the Sydney testnet. To ensure efficient debugging and testing, kindly adhere closely to the provided instructions. Please avoid reusing existing $USER and create new one exclusively. Your cooperation in this matter is greatly appreciated._

### Introduction:

In this tutorial, we will guide you through the installation of the GGX node using Systemd. However, please note that the security of your infrastructure is not addressed here. It is important to [follow best practices](https://www.debian.org/doc/manuals/securing-debian-manual/index.en.html) for ensuring the security of your system.

The primary objective of this setup is to establish a high-performance validator. As such, it is crucial to ensure that all system resources are exclusively dedicated to our application. Deploying the application within a Docker environment is generally not recommended as each virtualization layer introduces overhead and can diminish validator performance. For optimal performance, we highly recommend running only one validator per dedicated hardware instance.

#### A little guidance for choosing hardware:
* _Reasonable Modern Linux distro ( Debian, Ubuntu ) configurable accordingly_
* _AMD Zen3 or above, Intel Ice Lake, or newer (Xeon, Ryzen or Core series)_
* _4+ physical cores @ 3.4GHz or above_
* _Simultaneous multithreading disabled ( edge production )_
* _Prefer single-threaded performance over higher cores count_
* _Enterprise NVMe SSD 512GB+ ( the sizing needs to be proportionate to accommodate the growing size of the blockchain )_
* _16GB+ DDR4 ECC_
* _Latest Linux Kernel_
* _A minimum symmetric networking speed of 100Mb/s is required._

### Preparation:

_Assume we logged in as our admin user who is a member of sudo group, let's go_

```sh
# system upgrade
sudo apt update && sudo apt upgrade -y
```

* _Reboot server_

```sh
sudo apt install git wget curl jq -y
```

```sh
# Install requirements
sudo apt install build-essential protobuf-compiler pkg-config libssl-dev librust-clang-sys-dev -y
```

* **Create user**

This user should not be granted login privileges and should not be allowed to set any passwords.

```sh
# For example our user name is ggx_user
USER_NAME='ggx_user'
```

```sh
# Create dedicated no-login user
sudo adduser --disabled-login --disabled-password --gecos GECOS ${USER_NAME}
```

```sh
# get user shell ( stay here untill we ready to start the node )
sudo su - ${USER_NAME}
```

* **Set variables** _( Before integrating the parameters, kindly ensure the versions are up-to-date by performing a cross-check. )_

We will need `USER_NAME`, `NODE_SYSTEM_NAME` later, please note.

```sh
# Set Rust Toolchain and node binary version
# The entries below can be accidently left outdated and lead to unpredictable consequences
RUST_TOOLCHAIN='nightly-2023-08-19'
GGX_NODE_VERSION='v0.1.6'
# Below can be set based on your personal preferences
USER_NAME="$(whoami)"                   # process owner
NODE_SYSTEM_NAME='MyNodaName123'        # used for data folder name for easy identification
# Make sure no "." or any special characters is involved.
NODE_PRETTY_NAME='My Noda Sir 123'      # This will be visible on public telemetry dashboard
```

* **Rust toolchain and additional components**

```sh
# Install Rust default profile
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y \
     --default-toolchain ${RUST_TOOLCHAIN} \
     --profile default && source "$HOME/.cargo/env"
```

```sh
# Update toolchain and set default
rustup update  ${RUST_TOOLCHAIN}
rustup default ${RUST_TOOLCHAIN}
```

```sh
# Install required components
rustup target add wasm32-unknown-unknown --toolchain ${RUST_TOOLCHAIN}
rustup component add rust-src --toolchain ${RUST_TOOLCHAIN}
```

### Installation:

```sh
# Clone repository
cd ~ && git clone https://github.com/ggxchain/ggxnode.git
cd ggxnode && git fetch --all --tags && git pull
```

```sh
# Checkout required version ( please cross-check )
git checkout ${GGX_NODE_VERSION}
```

```sh
# Build ( Sydney Testnet ) This can take several hours, depends on the computer resources available.
rustup run ${RUST_TOOLCHAIN} cargo build --locked --release --package ggxchain-node --no-default-features --features="sydney"
```

If the build fails for any reason, please reach out to the community validators on [Dicord](https://discord.gg/ggx) for assistance.

### Server configuration

In order to ensure the provisioning of necessary resources, it is imperative to appropriately configure the system. The approach to achieving this will vary depending on the specific distribution, hardware specifications, kernel version, and other relevant factors. Attempting to provide a comprehensive and universally applicable guide in this context would be unfeasible. Therefore, we will present a concise checklist to facilitate the configuration process.

* Set CPU Governor to Performance
* Increase Max Open Files Limit
* Make sure this adjustments is preserver on boot

### Systemd

At the time of writing this documentation, the network is in the early testnet stage. Certain variables may undergo multiple changes. To streamline the adjustment of these parameters, we will create a configuration file and include it in the systemd unit configuration. This approach enables more coherent and organized management of these parameters in a transparent and readable manner, ensuring the freshness and accuracy of the values.

**Prior to deploying the node, it is crucial to thoroughly validate all essential parameters. Here is a handy checklist to ensure a smooth deployment:**

* _Node binary version_
* _Rust toolchain version_

_Good place to confirm this is GGXChain public [Dicord](https://discord.gg/ggx) server_

#### Folders & Names

* **_Please ensure that `${USER_NAME}` user has read and write permissions._**

```sh
# Create binary home
cd ~ && mkdir -p "${HOME}/bin"
```

```sh
# add $HOME/bin to $PATH ( bash )
echo 'export PATH="${HOME}/bin:$PATH"' >>.bashrc && . .bashrc
```

_At the time of writtining `we are on the Sydney` test network. However, can be set to your prefered location. Variable `${NODE_SYSTEM_NAME}` we set before is just example for better recognition, can be set according personal preference as `db`, `node_1` `data` or anything also._

```sh
# Create data folder aka BASE_PATH ( we will need this later, remember )
BASE_PATH="${HOME}/data-sydney/${NODE_SYSTEM_NAME}"
mkdir -p ${BASE_PATH} && chmod 0700 ${BASE_PATH}
```

```sh
# Folder path to store node key
PRIVATE_KEY_STORAGE_FOLDER="${HOME}/.private"
mkdir -p "${PRIVATE_KEY_STORAGE_FOLDER}" && chmod 0700 "${PRIVATE_KEY_STORAGE_FOLDER}"
```

#### Make symlink to previously compiled binary

```sh
ln -s ${HOME}/ggxnode/target/release/ggxchain-node ${HOME}/bin/
```

```sh
# Test
ggxchain-node --version
```

### Creating Config

**This step-by-step tutorial doesn't require any manual action, however just for better understanding of configuration file here is short description:**

* **node name**, some special characters will not be accepted as such as `.`
* **Path should be absolute**, double check if all locations _( created above )_ are in place.
* For _author_rotateKeys_ method we do need `RPC_METHODS` to be `unsafe`, after activation please set to `safe` and **restart**
* `BASE_PATH` is where database are stored. **Point to the same location we just choose previously**
* `CUSTOM_CHAIN_SPEC` is a chain name. We will use `sydney` in our example, as we about to join Sydney testnet.
* `NODE_KEY_FILE` you on your own on how to manage your `node.key`. Please follow best practices. Never stop research and improving security.
* `RPC_PORT` `PROMETHEUS_PORT` `CONSENSUS_P2P` are flexible and can be set according installation preferences.
* `SYNC_MODE` available methods `full` `warp` `fast` _( archive mod only support full sync )_

```sh
# Create folder to store configuration
mkdir -p ${HOME}/conf && chmod 0700 ${HOME}/conf
```

Execute command block below, make sure everything up to date, adjust prooning parameters if required.
`RPC_METHODS` will be set temporaty to `unsafe`, as we need to perform `keys_rotation call` required by **validator**.
If you setting up passive observer node, set this to `safe` now.

```bash
        # Create node configuration
        cat <<EOF | tee ${HOME}/conf/node.conf
RPC_METHODS=unsafe
NODE_NAME=${NODE_PRETTY_NAME}
BASE_PATH=/home/${USER_NAME}/data-sydney/${NODE_SYSTEM_NAME}
NODE_KEY_FILE=${PRIVATE_KEY_STORAGE_FOLDER}/node.key
CUSTOM_CHAIN_SPEC=sydney
RPC_PORT=9933
PROMETHEUS_PORT=9615
CONSENSUS_P2P=33777
RPC_CORS=localhost
LOG_LEVEL=info
STATE_PRUNING=archive
BLOCKS_PRUNING=archive
SYNC_MODE=full
EOF
```

_( this configuration file can be easily updated later though our journey `${HOME}/conf/node.conf` )_

```sh
# Restrict permissions
chmod 0600 ${HOME}/conf/node.conf
```

### Keys Generation

_Private keys are highly sensitive files and should be handled with utmost care. It is essential to ensure maximum protection and prevent any potential exposure. It is strongly recommended to follow best practices for keys management._ _Make sure to keep your private keys private and avoid sharing them publicly. Implement measures to safeguard the confidentiality and integrity of these keys. It is crucial to be aware of GGX Chain node keys management techniques to ensure secure handling._

```sh
# generate node key
ggxchain-node key generate-node-key --file "${PRIVATE_KEY_STORAGE_FOLDER}/node.key"
```

```sh
# set permissions
chmod 0600 "${PRIVATE_KEY_STORAGE_FOLDER}/node.key"
```

Check node public ID

```sh
# check node public ID
ggxchain-node key inspect-node-key --file "${PRIVATE_KEY_STORAGE_FOLDER}/node.key"
```

* Backup encrypt, save securely _( not recommended in production, consult our community for better picture )_
* Please note, as soon as you ask community regarding best security practices and private keys management, big chance what you will instantly receive multiple private messages. Remember, everyone who sent you private messages are scammers. Always communicate in public chats and avoid any private communications with anyone.

_By design, GGX Chain doesn't require `node.key` backup for security reason. But we are on testnet now, remember this._

### Create Systemd Unit Config

To continue we need **sudo** right, but just before exit lets copy our user name and node system name in to clipboard

```sh
# execute this command and follow instruction
echo -e "\n# Copy this lines below in to the clipboard: \nUSER_NAME=$(whoami)\nNODE_SYSTEM_NAME=${NODE_SYSTEM_NAME}" && echo
```

```sh
# Exit from current non-sudo user shell
exit
```

_Our wariables we set previously in user environment will be gone, lets set them again_

* Paste copied lines from the step above and press enter to set them for sudo user environment

```sh
# Check if variables set
echo " User name: ${USER_NAME}" && echo " Node system name: ${NODE_SYSTEM_NAME}"
```

Create systemd unit configuration

```sh
# Create systemd unit configuration
sudo cat <<EOF | sudo tee /etc/systemd/system/ggx-node.service > /dev/null
[Unit]
Description=GGXChain Node
Wants=network-online.target
After=network-online.target

[Service]
User=${USER_NAME}
Group=${USER_NAME}

Type=simple

EnvironmentFile=/home/${USER_NAME}/conf/node.conf

ExecStart=/home/${USER_NAME}/bin/ggxchain-node \\
  --port \${CONSENSUS_P2P} \\
  --base-path=\${BASE_PATH} \\
  --database rocksdb \\
  --sync \${SYNC_MODE} \\
  --no-private-ip \\
  --no-mdns \\
  --state-pruning \${STATE_PRUNING} \\
  --blocks-pruning \${BLOCKS_PRUNING} \\
  --node-key-type ed25519 \\
  --node-key-file \${NODE_KEY_FILE} \\
  --log \${LOG_LEVEL} \\
  --wasm-execution Compiled \\
  --rpc-methods \${RPC_METHODS} \\
  --rpc-cors "localhost" \\
  --rpc-port \${RPC_PORT} \\
  --prometheus-port \${PROMETHEUS_PORT} \\
  --name \${NODE_NAME} \\
  --chain \${CUSTOM_CHAIN_SPEC}

Restart=always
RestartSec=160
LimitNOFILE=280000

[Install]
WantedBy=multi-user.target
EOF

```

```sh
# Reload configuration
sudo systemctl daemon-reload && sudo systemctl enable ggx-node.service
```

### Start The Node !

```sh
sudo systemctl start ggx-node.service && sudo journalctl -fu ggx-node.service -o cat
```

* _Logs will populate console screen by now, follow `monitoring` and `debugging` section of the documentation from here._
* _Node also should be visible at ==> [Telemetry Page](https://telemetry.ggxchain.net) <==_

Feel free to experiment with parameters in `/bin/node.conf` before taking any serious actions.

**_A little summary of what we just deployed:_**

* `RPC`           _bond to host and exposed on port `$RPC_PORT`, set to `unsafe` or `safe`_
* `Prometheus`    _bond to host and exposed on port `$PROMETHEUS_PORT`_
* `Consensus P2P` _bond to all interfaces and available on port `$CONSENSUS_P2P`_

**Validator**

* _Currently, this node is passive observer, validator require additional flag to be passed here `/etc/systemd/system/ggx-node.service`_

```sh
        --validator
```

**Example:**
```sh
ExecStart=/home/${USER_NAME}/bin/ggxchain-node --port ${CONSENSUS_P2P} --validator \
... rest of configuration ...
```

```sh
# Ensure unit configuration is reload
sudo systemctl daemon-reload
```

* To perform the `author_rotateKeys`, execute:

```sh
curl -H "Content-Type: application/json" \
     -d '{"id":1, "jsonrpc":"2.0", "method": "author_rotateKeys", "params":[]}' \
      http://localhost:$RPC_PORT
```

* _After successful call, set `$RPC_METHODS` to `safe` and restart `ggx-node.service`_
* Perform required transactions by using [GGXChain Explorer](https://explorer.ggxchain.io) interface

**New validators will land in to the waiting list first, please check "Waiting" [explorer tab](https://explorer.ggxchain.io/#/staking) before asking questions!**

#### Protocol Upgrade

GGXChain provides seamless upgrades with no downtime for validators. We will add detailed coverage of this feature to our documentation portal when it becomes relevant.

**Have fun ! And if you think we can improve this documentation feel free to collaborate, [talk to us](https://discord.gg/ggx).**
