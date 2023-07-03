<p style="font-size:14px" align="right">
<a href="https://cryptech.com.ua/" target="_blank">Visit our website <img src="https://storage.googleapis.com/c-pictures/logo_6c33163af2_dbef49e129_d981e5eee8/logo_6c33163af2_dbef49e129_d981e5eee8.png" width="150"/></a>
</p>


# Archway node setup for mainnet — archway-1

Explorers:
>- https://archway.explorers.guru/
>- https://explorers.cryptech.com.ua/archway-mainnet

Our services:

RPC
>- https://rpc-archway.cryptech.com.ua

API
>- https://api-archway.cryptech.com.ua

GRPC
>- https://grpc-archway.cryptech.com.ua

## Hardware Requirements
Like any Cosmos-SDK chain, the hardware requirements are pretty modest.

### Hardware Requirements
 - x86-64 (amd64) multi-core CPU (AMD / Intel)
 - 64GB RAM
 - 1TB NVMe SSD Storage
 - 100Mbps bi-directional Internet connection

## Set up your Quicksilver testnet fullnode

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
echo "export ARCHWAY_CHAIN_ID=archway-1" >> $HOME/.bash_profile
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
  ver="1.20"
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
git clone https://github.com/archway-network/archway && cd archway
git checkout v1.0.0
make install
```

## Config app
```
archwayd config chain-id $ARCHWAY_CHAIN_ID
```

## Init app
```
archwayd init $NODENAME --chain-id $ARCHWAY_CHAIN_ID
```

## Download genesis
```
rm ~/.archway/config/genesis.json
wget https://github.com/archway-network/networks/raw/main/archway-1/genesis/genesis.json.gz
gzip -d genesis.json.gz
mv genesis.json ~/.archway/config/
```

## Set seeds and peers
```
SEEDS="3ba7bf08f00e228026177e9cdc027f6ef6eb2b39@35.232.234.58:26656"
PEERS=""
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.archway/config/config.toml
```

## Config pruning
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.archway/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.archway/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.archway/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.archway/config/app.toml
```

## Set minimum gas price
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.005uqck\"/" $HOME/.archway/config/app.toml
```

## Reset chain data
```
archwayd tendermint unsafe-reset-all --home $HOME/.archway
```

## Create service
```
sudo tee /etc/systemd/system/archwayd.service > /dev/null <<EOF
[Unit]
Description=Quicksilver
After=network-online.target

[Service]
User=$USER
ExecStart=$(which archwayd) start
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
sudo systemctl enable archwayd
sudo systemctl restart archwayd && sudo journalctl -u archwayd -f -o cat
```
## Post installation

When installation is finished please load variables into system
```
source $HOME/.bash_profile
```

Next you have to make sure your validator is syncing blocks. You can use command below to check synchronization status
```
archwayd status 2>&1 | jq .SyncInfo
```

### (OPTIONAL) Disable and cleanup indexing
```
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.archway/config/config.toml
sudo systemctl restart archwayd
sleep 3
sudo rm -rf $HOME/.archway/data/tx_index.db
```

### (OPTIONAL) State Sync
You can state sync your node in minutes by running commands below.
```
SNAP_RPC1="https://rpc-archway.cryptech.com.ua:443" \
&& SNAP_RPC2="https://rpc-archway.cryptech.com.ua:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC2/block | jq -r .result.block.header.height) \
&& BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)) \
&& TRUST_HASH=$(curl -s "$SNAP_RPC2/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC1,$SNAP_RPC2\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.archway/config/config.toml
archwayd tendermint unsafe-reset-all --home $HOME/.archway
sudo systemctl restart archwayd && journalctl -fu archwayd -o cat
```

### Create wallet
To create new wallet you can use command below. Don’t forget to save the mnemonic
```
archwayd keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase
```
archwayd keys add $WALLET --recover
```

To get current list of wallets
```
archwayd keys list
```

### Save wallet info
Add wallet and valoper address into variables 
```
ARCHWAY_WALLET_ADDRESS=$(archwayd keys show $WALLET -a)
```
```
ARCHWAY_VALOPER_ADDRESS=$(archwayd keys show $WALLET --bech val -a)
```
Load variables into the system
```
echo 'export ARCHWAY_WALLET_ADDRESS='${ARCHWAY_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export ARCHWAY_VALOPER_ADDRESS='${ARCHWAY_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Create validator
Before creating validator please make sure that you have at least 1 ARCH (1 ARCH is equal to 1000000000000000000 aarch) and your node is synchronized

To check your wallet balance:
```
archwayd query bank balances $ARCHWAY_WALLET_ADDRESS
```
> If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue 

To create your validator run command below
```
archwayd tx staking create-validator \
  --amount 1000000000000000000aarch \
  --from $WALLET \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.07" \
  --min-self-delegation "1" \
  --pubkey  $(archwayd tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $ARCHWAY_CHAIN_ID
```

## Security
To protect you keys please make sure you follow basic security rules

### Set up ssh keys for authentication
Good tutorial on how to set up ssh keys for authentication to your server can be found [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)

### Get list of validators
```
archwayd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

## Get currently connected peer list with ids
```
curl -sS http://localhost:26657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

## Usefull commands
### Service management
Check logs
```
journalctl -u archwayd -f -o cat
```

Start service
```
sudo systemctl start archwayd
```

Stop service
```
sudo systemctl stop archwayd
```

Restart service
```
sudo systemctl restart archwayd
```

### Node info
Synchronization info
```
archwayd status 2>&1 | jq .SyncInfo
```

Validator info
```
archwayd status 2>&1 | jq .ValidatorInfo
```

Node info
```
archwayd status 2>&1 | jq .NodeInfo
```

Show node id
```
archwayd tendermint show-node-id
```

### Wallet operations
List of wallets
```
archwayd keys list
```

Recover wallet
```
archwayd keys add $WALLET --recover
```

Delete wallet
```
archwayd keys delete $WALLET
```

Get wallet balance
```
archwayd query bank balances $ARCHWAY_WALLET_ADDRESS
```

Transfer funds
```
archwayd tx bank send $ARCHWAY_WALLET_ADDRESS <TO_ARCHWAY_WALLET_ADDRESS> <COST>aarch
```

### Voting
```
archwayd tx gov vote 1 yes --from $WALLET --chain-id=$ARCHWAY_CHAIN_ID
```

### Staking, Delegation and Rewards
Delegate stake
```
archwayd tx staking delegate $ARCHWAY_VALOPER_ADDRESS 1000000000000000000aarch --from=$WALLET --chain-id=$ARCHWAY_CHAIN_ID --gas=auto
```

Redelegate stake from validator to another validator
```
archwayd tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 1000000000000000000aarch --from=$WALLET --chain-id=$ARCHWAY_CHAIN_ID --gas=auto
```

Withdraw all rewards
```
archwayd tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$ARCHWAY_CHAIN_ID --gas=auto
```

Withdraw rewards with commision
```
archwayd tx distribution withdraw-rewards $ARCHWAY_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=$ARCHWAY_CHAIN_ID
```

### Validator management
Edit validator
```
archwayd tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=$ARCHWAY_CHAIN_ID \
  --from=$WALLET
```

Unjail validator
```
archwayd tx slashing unjail \
  --broadcast-mode=block \
  --from=$WALLET \
  --chain-id=$ARCHWAY_CHAIN_ID \
  --gas=auto
```

### Delete node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop archwayd
sudo systemctl disable archwayd
sudo rm /etc/systemd/system/archway* -rf
sudo rm $(which archwayd) -rf
sudo rm $HOME/.archway* -rf
sudo rm $HOME/archway -rf
sed -i '/ARCHWAY_/d' ~/.bash_profile
```
