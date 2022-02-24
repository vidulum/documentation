# Mainnet Full Node & Validator

<!-- MarkdownTOC autolink="true" -->

- [SECTION 0: Requirements](#section-0-requirements)
- [SECTION 1: System preparation](#section-1-system-preparation)
  - [Add dedicated user](#add-dedicated-user)
  - [Go deployment](#go-deployment)
    - [Download and extract repository](#download-and-extract-repository)
  - [Firewall Configuration](#firewall-configuration)
  - [systemd Service Configuration](#systemd-service-configuration)
- [SECTION 2: Build and Initiate Vidulum Node](#section-2-build-and-initiate-vidulum-node)
  - [Add Go environmental variables](#add-go-environmental-variables)
  - [Build Vidulum binaries](#build-vidulum-binaries)
  - [Vidulum Node Init](#vidulum-node-init)
  - [Start node](#start-node)
- [SECTION 3: Promote Full Node to Validator Role](#section-3-promote-full-node-to-validator-role)
  - [Create Wallet](#create-wallet)
  - [Create Validator](#create-validator)

<!-- /MarkdownTOC -->

## SECTION 0: Requirements

- Ubuntu 20.04 LTS

**Minimum**

- 2 CPUs
- 4GB RAM
- 200GB SSD

**Recommended**

- 4 CPUs
- 8GB RAM
- 1TB SSD

## SECTION 1: System preparation

::: warning NOTE:
All tasks in **SECTION 1** have to be performed as **root**
:::

### Add dedicated user

```bash
sudo adduser vidulum
```
### Install prerequisites

Update and install *make*, which is needed to compile vidulum:
```bash
apt-get update
apt-get install -y build-essential
```

### Go deployment

#### Download and extract repository

```bash
GOVER=$(curl https://go.dev/VERSION?m=text)
wget https://golang.org/dl/${GOVER}.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf ${GOVER}.linux-amd64.tar.gz
```

**NOTE**: That will install latest version of Go

### Firewall Configuration

```bash
ufw limit ssh/tcp comment 'Rate limit for openssh server'
ufw default deny incoming
ufw default allow outgoing
ufw allow 26656/tcp comment 'Cosmos SDK/Tendermint P2P (Vidulum Validator)'
ufw enable
```

### systemd Service Configuration

Create a service file **_/lib/systemd/system/vidulum.service_** 
```bash
nano /lib/systemd/system/vidulum.service
```

Paste the following text into the editor and save the file:

```bash
[Unit]
Description=Vidulum Validator
After=network.target

[Service]
Group=vidulum
User=vidulum
WorkingDirectory=/home/vidulum
ExecStart=/home/vidulum/.local/bin/vidulumd start
Restart=on-failure
RestartSec=3
LimitNOFILE=8192

[Install]
WantedBy=multi-user.target
```

Reload the systemd configuration and enable the service.

```bash
systemctl daemon-reload && systemctl enable vidulum.service
```

## SECTION 2: Build and Initiate Vidulum Node

::: warning NOTE:
All tasks in **SECTION 2** must be performed as the **vidulum** user created in **SECTION 1**
:::

### Add Go environmental variables

Set of variables, which should be set for user(s) with the need to build Go Applications.

Add Golang specific variables to `${HOME}/.profile`

```bash
# add environmental variables for Go
if [ -f "/usr/local/go/bin/go" ] ; then
    export GOROOT=/usr/local/go
    export GOPATH=${HOME}/go
    export GOBIN=$GOPATH/bin
    export PATH=${PATH}:${GOROOT}/bin:${GOBIN}
fi
```

Once modified and saved, reload `${HOME}/.profile` to set variables in the current user session:

```bash
. ~/.profile
```

### Build Vidulum binaries

Before we build the binaries for a Vidulum node/validator, create a folder where the binaries will be stored.
Ubuntu adds this folder to search path, when it exists, so we can easily run binaries in future when needed.

```bash
mkdir -p ${HOME}/.local/bin
. ~/.profile
```

Clone the Vidulum GitHub repository, build the binaries, and move it to your users local bin directory.

```bash
git clone https://github.com/vidulum/mainnet && cd mainnet && make install
mv ${HOME}/go/bin/vidulumd ${HOME}/.local/bin
```

### Vidulum Node Init

```bash
vidulumd init NODE_NAME --chain-id vidulum-1
```

::: tip NOTE:
Replace NODE_NAME with the name you want to assign to your validator.
:::

In **_\${HOME}/.vidulum/config/config.toml_** find the **_[p2p]_** section, and change the following to match:

```bash
seeds = "883ec7d5af7222c206674c20c997ccc5c242b38b@ec2-3-82-120-39.compute-1.amazonaws.com:26656,eed11fff15b1eca8016c6a0194d86e4a60a65f9b@apollo.erialos.me:26656"
```

Now find the **[statesync]** section and change the config to match the following highlighted lines:

```bash{13,25-29}
#######################################################
###         State Sync Configuration Options        ###
#######################################################
[statesync]
# State sync rapidly bootstraps a new node by discovering, fetching, and restoring a state machine
# snapshot from peers instead of fetching and replaying historical blocks. Requires some peers in
# the network to take and serve state machine snapshots. State sync is not attempted if the node
# has any local state (LastBlockHeight > 0). The node will have a truncated block history,
# starting from the height of the snapshot.
enable = true

# RPC servers (comma-separated) for light client verification of the synced state machine and
# retrieval of state data for node bootstrapping. Also needs a trusted height and corresponding
# header hash obtained from a trusted source, and a period during which validators can be trusted.
#
# For Cosmos SDK-based chains, trust_period should usually be about 2/3 of the unbonding time (~2
# weeks) during which they can be financially punished (slashed) for misbehavior.
rpc_servers = "https://trpc.rpc.erialos.me:443,https://mainnet-rpc.vidulum.app:443"
trust_height = 1483100
trust_hash = "F9E4CFBC5FA0DB49CF43563244391780CF8BFE5B34CA95B3DEE22774418D315A"
trust_period = "336h0m0s"
#21 days unbonding - 2/3 time is 14 days or 336 hours.

# Time to spend discovering snapshots before initiating a restore.
discovery_time = "15s"

# Temporary directory for state sync snapshot chunks, defaults to the OS tempdir (typically /tmp).
# Will create a new, randomly named directory within, and remove it when done.
temp_dir = ""

# The timeout duration before re-requesting a chunk, possibly from a different
# peer (default: 1 minute).
chunk_request_timeout = "10s"

# The number of concurrent chunk fetchers to run (default: 1).
chunk_fetchers = "4"
```

::: tip TIP:
Don't use the above height! Get a recent block height and header hash by visiting the explorer! Snapshots are taken at 100 block height intervals. 
:::

![blockheigh-hash-statesync](~@source/getting-started/assets/blockheight-hash-statesync.png)

Now it is time to download **_genesis.json_** file, which will allow the node to synchronize

```bash
wget https://raw.githubusercontent.com/vidulum/mainnet/main/genesis.json -O ${HOME}/.vidulum/config/genesis.json
```

### Start node

Once the node is configured, you can start it, and begin to synchronize the blockchain database.

```bash
sudo systemctl start vidulum.service
```

To keep watching logs generated by Vidulum node use the command:

```bash
journalctl -u vidulum -f
```

Envoke watch with curl to check the status every 60 seconds if node has finished **catching up** and has fully synchronized the blockchain:

```bash
watch -n 60 "curl http://localhost:26657/status"
```

You will see JSON output where you need to locate **_catching_up_** field. When it will have value **_true_** means node is still synchronizing. When value is **_false_** means node is fully synchronized. Then you can move on to creating validator.

```json{15}
........
    },
    "sync_info": {
      "latest_block_hash": "39B4F3361E27EEC32605DA2F554FE2C64EAB41A78B593531CC5E11EEDE9AD67C",
      "latest_app_hash": "05E9931EAF0284B4024D5393B8878146C1CFE0329183EE6084BD4EF48507FA29",
      "latest_block_height": "544241",
      "latest_block_time": "2021-11-29T20:26:22.352845323Z",
      "earliest_block_hash": "5614A5F7D398CCAC49EFB255D1F92421891B725808E31421317BDD519D35F7CA",
      "earliest_app_hash": "E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855",
      "earliest_block_height": "1",
      "earliest_block_time": "2021-10-23T00:00:00Z",
      "catching_up": false
    },
........
```

## SECTION 3: Promote Full Node to Validator Role

::: warning NOTE:
All tasks in **SECTION 3** have to be performed as the **vidulum** user created in **SECTION 1**.<br>
**ONLY** continue this section if you have plans to impliment a validator node. 
:::

In order to create a validator, you must have a Vidulum wallet and at least 1 VDL that you must self delegate as the validator.

### Create Wallet

In order to create a Vidulum wallet we use binaries we compiled earlier.

```bash
vidulumd keys add WALLET_NAME --keyring-backend os
```

You will be asked to provide a password.  You will use this to claim rewards, commission, and vote.

Output of this command will be similar to presented below

```bash
- name: WALLET_NAME
  type: local
  address: vdl1hjhglrzggqtdhsh3ag8jp0cckmva5pe976jxel
  pubkey: '{"@type":"/cosmos.crypto.secp256k1.PubKey","key":"Anriv0TNrt1cz3+pSq2UDNiJQZINNlgtknousVlcujZ7"}'
  mnemonic: ""


**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

some words forming mnemonic seed will be placed here you have to write them down and keep them safe
```

::: tip NOTE:
Write or print out your `mnemonic seed` and keep it in a safe place if you ever need to restore your validator's wallet.
:::

Now you have to transfer some vidulum to your validator wallet. To check the balance on your account:

```bash
vidulumd query bank balances vdl1hjhglrzggqtdhsh3ag8jp0cckmva5pe976jxel
```

The output will be similar to this:

```bash
balances:
- amount: "15000000000"
  denom: uvdl
pagination:
  next_key: null
  total: "0"
```

::: tip NOTE:
Denomiation presented by command is in uvdl(micro-vidulum). Use this formula to convert, 1 VDL = 1,000,000 uVDL.
:::

### Create Validator

Once you have funds in your validator wallet, promote your full node to a validator:

```bash
vidulumd tx staking create-validator \
    --commission-max-change-rate="0.05" \
    --commission-max-rate="0.3" \
    --commission-rate="0.11" \
    --amount="10000000uvdl" \
    --pubkey=$(vidulumd tendermint show-validator) \
    --website="https://your.website" \
    --details="Description of your validator." \
    --security-contact="contact@your.domain" \
    --moniker=NODE_NAME \
    --chain-id=vidulum-1 \
    --min-self-delegation="1" \
    --gas auto \
    --gas-adjustment=1.2 \
    --fees 200000uvdl \
    --from=WALLET_NAME \
    --keyring-backend os
```

::: tip NOTE:
The commission max rate cannot be changed once set! <br>
The commission rate may be changed *once* daily.
:::

Once that is done, wait a few moments, and you should see your node listed in [Vidulum Explorer](https://explorers.vidulum.app/vidulum/staking)<br>
Or [Ping.Pub Vidulum Explorer](https://ping.pub/vidulum/uptime)
