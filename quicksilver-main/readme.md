<p style="font-size:14px" align="right">
<a href="https://cryptech.com.ua/" target="_blank">Visit our website <img src="https://storage.googleapis.com/c-pictures/logo_6c33163af2_dbef49e129_d981e5eee8/logo_6c33163af2_dbef49e129_d981e5eee8.png" width="150"/></a>
</p>


# Quicksilver node setup for mainnet — quicksilver-2

Explorers:
>- https://quicksilver.explorers.guru/
>- https://explorers.cryptech.com.ua/quicksilver

Our services:

RPC
>- https://rpc-quicksilver-main.cryptech.com.ua

API
>- https://api-quicksilver-main.cryptech.com.ua

GRPC
>- https://grpc-quicksilver-main.cryptech.com.ua

## Hardware Requirements
Like any Cosmos-SDK chain, the hardware requirements are pretty modest.

### Minimum Hardware Requirements
 - 4x CPUs; the faster clock speed the better
 - 16GB RAM
 - 500GB Disk
 - Permanent Internet connection (traffic will be minimal during testnet; 100Mbps will be plenty - for production at least 100Mbps is expected)

### Recommended Hardware Requirements 
 - 8x CPUs; the faster clock speed the better
 - 32GB RAM
 - 1000GB of storage (SSD or NVME)
 - Permanent Internet connection 1000Mbps

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
echo "export QUICKSILVER_CHAIN_ID=quicksilver-2" >> $HOME/.bash_profile
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
  ver="1.19"
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
git clone https://github.com/ingenuity-build/quicksilver && cd quicksilver
git checkout v1.2.13
make install
```

## Config app
```
quicksilverd config chain-id $QUICKSILVER_CHAIN_ID
```

## Init app
```
quicksilverd init $NODENAME --chain-id $QUICKSILVER_CHAIN_ID
```

## Download genesis
```
rm ~/.quicksilverd/config
curl -s https://raw.githubusercontent.com/ingenuity-build/mainnet/main/genesis.json > ~/.quicksilver/config/genesis.json
```

## Set seeds and peers
```
SEEDS="20e1000e88125698264454a884812746c2eb4807@seeds.lavenderfive.com:11156,babc3f3f7804933265ec9c40ad94f4da8e9e0017@seed.rhinostake.com:11156,00f51227c4d5d977ad7174f1c0cea89082016ba2@seed-quick-mainnet.moonshot.army:26650"
PEERS="9f5751f22f485a56cf28b55d7b5c9a196a469f91@185.144.99.20:36656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.quicksilverd/config/config.toml
```

## Config pruning
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.aura/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.quicksilverd/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.quicksilverd/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.quicksilverd/config/app.toml
```

## Set minimum gas price
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.005uqck\"/" $HOME/.quicksilverd/config/app.toml
```

## Reset chain data
```
quicksivlerd tendermint unsafe-reset-all --home $HOME/.quicksilverd
```

## Create service
```
sudo tee /etc/systemd/system/quicksilverd.service > /dev/null <<EOF
[Unit]
Description=Quicksilver
After=network-online.target

[Service]
User=$USER
ExecStart=$(which quicksilverd) start
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
sudo systemctl enable quicksilverd
sudo systemctl restart quicksilverd && sudo journalctl -u quicksilverd -f -o cat
```
## Post installation

When installation is finished please load variables into system
```
source $HOME/.bash_profile
```

Next you have to make sure your validator is syncing blocks. You can use command below to check synchronization status
```
quicksilverd status 2>&1 | jq .SyncInfo
```

### (OPTIONAL) Disable and cleanup indexing
```
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.quicksilverd/config/config.toml
sudo systemctl restart quicksilverd
sleep 3
sudo rm -rf $HOME/.quicksilverd/data/tx_index.db
```

### (OPTIONAL) State Sync
You can state sync your node in minutes by running commands below.
```
SNAP_RPC1="https://rpc-quicksilverd-main.cryptech.com.ua:443" \
&& SNAP_RPC2="https://rpc-quicksilverd-main.cryptech.com.ua:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC2/block | jq -r .result.block.header.height) \
&& BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)) \
&& TRUST_HASH=$(curl -s "$SNAP_RPC2/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC1,$SNAP_RPC2\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.quicksilverd/config/config.toml
quicksilverd tendermint unsafe-reset-all --home $HOME/.quicksilverd
sudo systemctl restart quicksilverd && journalctl -fu quicksilverd -o cat
```

### Create wallet
To create new wallet you can use command below. Don’t forget to save the mnemonic
```
quicksilverd keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase
```
quicksilverd keys add $WALLET --recover
```

To get current list of wallets
```
quicksilverd keys list
```

### Save wallet info
Add wallet and valoper address into variables 
```
QUICKSILVER_WALLET_ADDRESS=$(quicksilverd keys show $WALLET -a)
```
```
QUICKSILVER_VALOPER_ADDRESS=$(quicksilverd keys show $WALLET --bech val -a)
```
Load variables into the system
```
echo 'export QUICKSILVER_WALLET_ADDRESS='${QUICKSILVER_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export QUICKSILVER_VALOPER_ADDRESS='${QUICKSILVER_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Create validator
Before creating validator please make sure that you have at least 1 QCK (1 QCK is equal to 1000000 uqck) and your node is synchronized

To check your wallet balance:
```
quicksilverd query bank balances $QUICKSILVER_WALLET_ADDRESS
```
> If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue 

To create your validator run command below
```
quicksilverd tx staking create-validator \
  --amount 1000000uqck \
  --from $WALLET \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.07" \
  --min-self-delegation "1" \
  --pubkey  $(quicksilverd tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $QUICKSILVER_CHAIN_ID
```

## Security
To protect you keys please make sure you follow basic security rules

### Set up ssh keys for authentication
Good tutorial on how to set up ssh keys for authentication to your server can be found [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)

### Get list of validators
```
quicksilverd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

## Get currently connected peer list with ids
```
curl -sS http://localhost:26657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

## Usefull commands
### Service management
Check logs
```
journalctl -u quicksilverd -f -o cat
```

Start service
```
sudo systemctl start quicksilverd
```

Stop service
```
sudo systemctl stop quicksilverd
```

Restart service
```
sudo systemctl restart quicksilverd
```

### Node info
Synchronization info
```
quicksilverd status 2>&1 | jq .SyncInfo
```

Validator info
```
quicksilverd status 2>&1 | jq .ValidatorInfo
```

Node info
```
quicksilverd status 2>&1 | jq .NodeInfo
```

Show node id
```
quicksilverd tendermint show-node-id
```

### Wallet operations
List of wallets
```
quicksilverd keys list
```

Recover wallet
```
quicksilverd keys add $WALLET --recover
```

Delete wallet
```
quicksilverd keys delete $WALLET
```

Get wallet balance
```
quicksilverd query bank balances $QUICKSILVER_WALLET_ADDRESS
```

Transfer funds
```
quicksilverd tx bank send $QUICKSILVER_WALLET_ADDRESS <TO_QUICKSILVER_WALLET_ADDRESS> 10000000uqck
```

### Voting
```
quicksilverd tx gov vote 1 yes --from $WALLET --chain-id=$QUICKSILVER_CHAIN_ID
```

### Staking, Delegation and Rewards
Delegate stake
```
quicksilverd tx staking delegate $QUICKSILVER_VALOPER_ADDRESS 10000000uqck --from=$WALLET --chain-id=$QUICKSILVER_CHAIN_ID --gas=auto
```

Redelegate stake from validator to another validator
```
quicksilverd tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 10000000ueaura --from=$WALLET --chain-id=$QUICKSILVER_CHAIN_ID --gas=auto
```

Withdraw all rewards
```
quicksilverd tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$QUICKSILVER_CHAIN_ID --gas=auto
```

Withdraw rewards with commision
```
quicksilverd tx distribution withdraw-rewards $QUICKSILVER_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=$QUICKSILVER_CHAIN_ID
```

### Validator management
Edit validator
```
quicksilverd tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=$QUICKSILVER_CHAIN_ID \
  --from=$WALLET
```

Unjail validator
```
quicksilverd tx slashing unjail \
  --broadcast-mode=block \
  --from=$WALLET \
  --chain-id=$QUICKSILVER_CHAIN_ID \
  --gas=auto
```

### Delete node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop quicksilverd
sudo systemctl disable quicksilverd
sudo rm /etc/systemd/system/quicksilver* -rf
sudo rm $(which quicksilverd) -rf
sudo rm $HOME/.quicksilver* -rf
sudo rm $HOME/quicksilver -rf
sed -i '/QUICKSILVER_/d' ~/.bash_profile
```
