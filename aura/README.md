<p style="font-size:14px" align="right">
<a href="https://cryptech.com.ua/" target="_blank">Visit our website <img src="https://storage.googleapis.com/c-pictures/logo_6c33163af2_dbef49e129_d981e5eee8/logo_6c33163af2_dbef49e129_d981e5eee8.png" width="150"/></a>
</p>

<p align="center">
  <img height="100" height="auto" src="https://user-images.githubusercontent.com/50621007/177979901-4ac785e2-08c3-4d61-83df-b451a2ed9e68.png">
</p>

# Aura node setup for testnet — euphoria-2

Official documentation:
>- [Validator setup instructions](https://docs.aura.app/run-a-node)

Explorers:
>- https://euphoria.aurascan.io/validators
>- https://explorers.cryptech.com.ua/aura_euphoria-2

Our services:

RPC
>- https://rpc-aura.cryptech.com.ua

API
>- https://api-aura.cryptech.com.ua

GRPC
>- https://grpc-aura.cryptech.com.ua

## Hardware Requirements
Like any Cosmos-SDK chain, the hardware requirements are pretty modest.

### Minimum Hardware Requirements
 - 3x CPUs; the faster clock speed the better
 - 4GB RAM
 - 80GB Disk
 - Permanent Internet connection (traffic will be minimal during testnet; 10Mbps will be plenty - for production at least 100Mbps is expected)

### Recommended Hardware Requirements 
 - 4x CPUs; the faster clock speed the better
 - 8GB RAM
 - 200GB of storage (SSD or NVME)
 - Permanent Internet connection (traffic will be minimal during testnet; 10Mbps will be plenty - for production at least 100Mbps is expected)

## Set up your aura fullnode

## Setting up vars
Here you have to put name of your moniker (validator) that will be visible in explorer
```
NODENAME=<YOUR_MONIKER_NAME_GOES_HERE>
```

Save and import variables into system
```
echo "export NODENAME=$NODENAME" >> $HOME/.bash_profile
if [ ! $WALLET ]; then
	echo "export WALLET=wallet" >> $HOME/.bash_profile
fi
echo "export AURA_CHAIN_ID=euphoria-2" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## Update packages
```
sudo apt update && sudo apt upgrade -y
```

## Install dependencies
```
sudo apt install curl build-essential git wget jq make gcc tmux chrony -y
```

## Install go
```
if ! [ -x "$(command -v go)" ]; then
  ver="1.18.2"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```

## Download and build binaries
```
git clone --branch euphoria_v0.4.2 https://github.com/aura-nw/aura && cd aura
make install
```

## Config app
```
aurad config chain-id $AURA_CHAIN_ID
aurad config keyring-backend test
```

## Init app
```
aurad init $NODENAME --chain-id $AURA_CHAIN_ID
```

## Download genesis and addrbook
```
wget https://github.com/aura-nw/testnets/raw/main/euphoria-2/euphoria-2-genesis.tar.gz
tar -xvzf euphoria-2-genesis.tar.gz -C $HOME/.aura/config
```

## Set seeds and peers
```
SEEDS="705e3c2b2b554586976ed88bb27f68e4c4176a33@13.250.223.114:26656,b9243524f659f2ff56691a4b2919c3060b2bb824@13.214.5.1:26656"
PEERS="bfef15bb8b4cbc4fb777aa33e75e6064cc1ba5bf@185.144.99.14:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.aura/config/config.toml
```

## Config pruning
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.aura/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.aura/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.aura/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.aura/config/app.toml
```

## Set minimum gas price
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0ueaura\"/" $HOME/.aura/config/app.toml
```

## Enable prometheus
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.aura/config/config.toml
```

## Reset chain data
```
aurad unsafe-reset-all --home $HOME/.aura
```

## Create service
```
sudo tee /etc/systemd/system/aurad.service > /dev/null <<EOF
[Unit]
Description=aura
After=network-online.target

[Service]
User=$USER
ExecStart=$(which aurad) start --home $HOME/.aura
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Register and start service
```
sudo systemctl daemon-reload
sudo systemctl enable aurad
sudo systemctl restart aurad && sudo journalctl -u aurad -f -o cat
```
## Post installation

When installation is finished please load variables into system
```
source $HOME/.bash_profile
```

Next you have to make sure your validator is syncing blocks. You can use command below to check synchronization status
```
aurad status 2>&1 | jq .SyncInfo
```

### (OPTIONAL) Disable and cleanup indexing
```
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.aura/config/config.toml
sudo systemctl restart aurad
sleep 3
sudo rm -rf $HOME/.aura/data/tx_index.db
```

### (OPTIONAL) State Sync
You can state sync your node in minutes by running commands below.
```
SNAP_RPC1="https://snapshot-1.euphoria.aura.network:443" \
&& SNAP_RPC2="https://rpc.cryptech.com.ua:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC2/block | jq -r .result.block.header.height) \
&& BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000)) \
&& TRUST_HASH=$(curl -s "$SNAP_RPC2/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC1,$SNAP_RPC2\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.aura/config/config.toml
aurad tendermint unsafe-reset-all --home $HOME/.aura
sudo systemctl restart aurad && journalctl -fu aurad -o cat
```

### Create wallet
To create new wallet you can use command below. Don’t forget to save the mnemonic
```
aurad keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase
```
aurad keys add $WALLET --recover
```

To get current list of wallets
```
aurad keys list
```

### Save wallet info
Add wallet and valoper address into variables 
```
AURA_WALLET_ADDRESS=$(aurad keys show $WALLET -a)
```
```
AURA_VALOPER_ADDRESS=$(aurad keys show $WALLET --bech val -a)
```
Load variables into the system
```
echo 'export AURA_WALLET_ADDRESS='${AURA_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export AURA_VALOPER_ADDRESS='${AURA_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Create validator
Before creating validator please make sure that you have at least 1 kuji (1 kuji is equal to 1000000 ueaura) and your node is synchronized

To check your wallet balance:
```
aurad query bank balances $AURA_WALLET_ADDRESS
```
> If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue 

To create your validator run command below
```
aurad tx staking create-validator \
  --amount 100000ueaura \
  --from $WALLET \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.07" \
  --min-self-delegation "1" \
  --pubkey  $(aurad tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $AURA_CHAIN_ID
```

## Security
To protect you keys please make sure you follow basic security rules

### Set up ssh keys for authentication
Good tutorial on how to set up ssh keys for authentication to your server can be found [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)

### Get list of validators
```
aurad q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

## Get currently connected peer list with ids
```
curl -sS http://localhost:26657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

## Usefull commands
### Service management
Check logs
```
journalctl -u aurad -f -o cat
```

Start service
```
sudo systemctl start aurad
```

Stop service
```
sudo systemctl stop aurad
```

Restart service
```
sudo systemctl restart aurad
```

### Node info
Synchronization info
```
aurad status 2>&1 | jq .SyncInfo
```

Validator info
```
aurad status 2>&1 | jq .ValidatorInfo
```

Node info
```
aurad status 2>&1 | jq .NodeInfo
```

Show node id
```
aurad tendermint show-node-id
```

### Wallet operations
List of wallets
```
aurad keys list
```

Recover wallet
```
aurad keys add $WALLET --recover
```

Delete wallet
```
aurad keys delete $WALLET
```

Get wallet balance
```
aurad query bank balances $AURA_WALLET_ADDRESS
```

Transfer funds
```
aurad tx bank send $AURA_WALLET_ADDRESS <TO_AURA_WALLET_ADDRESS> 10000000ueaura
```

### Voting
```
aurad tx gov vote 1 yes --from $WALLET --chain-id=$AURA_CHAIN_ID
```

### Staking, Delegation and Rewards
Delegate stake
```
aurad tx staking delegate $AURA_VALOPER_ADDRESS 10000000ueaura --from=$WALLET --chain-id=$AURA_CHAIN_ID --gas=auto
```

Redelegate stake from validator to another validator
```
aurad tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 10000000ueaura --from=$WALLET --chain-id=$AURA_CHAIN_ID --gas=auto
```

Withdraw all rewards
```
aurad tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$AURA_CHAIN_ID --gas=auto
```

Withdraw rewards with commision
```
aurad tx distribution withdraw-rewards $AURA_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=$AURA_CHAIN_ID
```

### Validator management
Edit validator
```
aurad tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=$AURA_CHAIN_ID \
  --from=$WALLET
```

Unjail validator
```
aurad tx slashing unjail \
  --broadcast-mode=block \
  --from=$WALLET \
  --chain-id=$AURA_CHAIN_ID \
  --gas=auto
```

### Delete node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop aurad
sudo systemctl disable aurad
sudo rm /etc/systemd/system/aura* -rf
sudo rm $(which aurad) -rf
sudo rm $HOME/.aura* -rf
sudo rm $HOME/aura -rf
sed -i '/AURA_/d' ~/.bash_profile
```
