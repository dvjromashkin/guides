<p style="font-size:14px" align="right">
<a href="https://cryptech.com.ua/" target="_blank">Visit our website <img src="https://storage.googleapis.com/c-pictures/logo_6c33163af2_dbef49e129_d981e5eee8/logo_6c33163af2_dbef49e129_d981e5eee8.png" width="150"/></a>
</p>


# Celestia consensus node setup for testnet — mocha-3

Explorers:
>- https://celestia.explorers.guru/
>- https://explorers.cryptech.com.ua/celestia-mocha/staking

Our services:

RPC
>- https://rpc-celestia.cryptech.com.ua

API
>- https://api-celestia.cryptech.com.ua

GRPC
>- https://grpc-celestia.cryptech.com.ua

## Hardware Requirements
Like any Cosmos-SDK chain, the hardware requirements are pretty modest.
Mandatory requirement for installation - GO ver =<1.20

### Hardware Requirements
 - Memory: 8 GB RAM
 - CPU: 6 cores
 - Disk: 500 GB SSD Storage
 - Bandwidth: 1 Gbps for Download/1 Gbps for Upload

## Set up your Celestia testnet (mocha-3) fullnode

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
echo "export CELESTIA_CHAIN_ID=mocha-3" >> $HOME/.bash_profile
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
  ver="1.20.5"
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
cd $HOME
rm -rf celestia-app
git clone https://github.com/celestiaorg/celestia-app.git
cd celestia-app
git checkout v1.0.0-rc7
make build
make install
```

## Config app
```
celestia-appd config chain-id $CELESTIA_CHAIN_ID
```

## Init app
```
celestia-appd init $NODENAME --chain-id $CELESTIA_CHAIN_ID
```

## Download genesis
```
rm ~/.celestia-app/config/genesis.json
cd $HOME/.celestia-app/config
wget https://raw.githubusercontent.com/celestiaorg/networks/master/mocha-3/genesis.json
```

## Set seeds and peers
```
SEEDS="3314051954fc072a0678ec0cbac690ad8676ab98@65.108.66.220:26656"
PEERS="ec11f3be74010b78882de2cbd170d7ad4458d8ac@157.245.250.63:26656"
sed -i -e 's|^seeds *=.*|seeds = "'$SEEDS'"|; s|^persistent_peers *=.*|persistent_peers = "'$PEERS'"|' $HOME/.celestia-app/config/config.toml
sed -i -e "s/^seed_mode *=.*/seed_mode = \"$SEED_MODE\"/" $HOME/.celestia-app/config/config.toml
```

## Config pruning
```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="10"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.celestia-app/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.celestai-app/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.celestia-app/config/app.toml
```

## Set minimum gas price
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.005utia\"/" $HOME/.celestia-app/config/app.toml
```

## Reset chain data
```
celestia-appd tendermint unsafe-reset-all --home $HOME/.celestia-app
```

## Create service
```
sudo tee /etc/systemd/system/celestia-appd.service > /dev/null <<EOF
[Unit]
Description=Celestia-APP fullnode
After=network-online.target

[Service]
User=$USER
ExecStart=$(which celestia-appd) start
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
sudo systemctl enable celestia-appd
sudo systemctl restart celestia-appd && sudo journalctl -u celestia-appd -f -o cat
```
## Post installation

When installation is finished please load variables into system
```
source $HOME/.bash_profile
```

Next you have to make sure your validator is syncing blocks. You can use command below to check synchronization status
```
celestia-appd status 2>&1 | jq .SyncInfo
```

### (OPTIONAL) Disable and cleanup indexing
```
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.celestia-app/config/config.toml
sudo systemctl restart celestia-appd
sleep 3
sudo rm -rf $HOME/.celestia-app/data/tx_index.db
```

### (OPTIONAL) State Sync
You can state sync your node in minutes by running commands below.
```
SNAP_RPC1="https://rpc-celestia.cryptech.com.ua:443" \
&& SNAP_RPC2="https://rpc-celestia.cryptech.com.ua:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC2/block | jq -r .result.block.header.height) \
&& BLOCK_HEIGHT=$((LATEST_HEIGHT - 1000)) \
&& TRUST_HASH=$(curl -s "$SNAP_RPC2/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC1,$SNAP_RPC2\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"|" $HOME/.celestia-app/config/config.toml
celestia-appd tendermint unsafe-reset-all --home $HOME/.celestia-app
sudo systemctl restart celestia-appd && journalctl -fu celestia-appd -o cat
```

### Create wallet
To create new wallet you can use command below. Don’t forget to save the mnemonic
```
celestia-appd keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase
```
celestia-appd keys add $WALLET --recover
```

To get current list of wallets
```
celestia-appd keys list
```

### Save wallet info
Add wallet and valoper address into variables 
```
CELESTIA_WALLET_ADDRESS=$(celestia-appd keys show $WALLET -a)
```
```
CELESTIA_VALOPER_ADDRESS=$(celestia-appd keys show $WALLET --bech val -a)
```
Load variables into the system
```
echo 'export CELESTIA_WALLET_ADDRESS='${CELESTIA_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export CELESTIA_VALOPER_ADDRESS='${CELESTIA_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

### Create validator
Before creating validator please make sure that you have at least 1 TIA (1 TIA is equal to 1000000 utia) and your node is synchronized

To check your wallet balance:
```
celestia-appd query bank balances $CELESTIA_WALLET_ADDRESS
```
> If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue 

To create your validator run command below
```
celestia-appd tx staking create-validator \
  --amount 1000000utia \
  --from $WALLET \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.07" \
  --min-self-delegation "1" \
  --pubkey  $(celestia-appd tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $CELESTIA_CHAIN_ID
  --gas 250000 \
  --fees 21000utia \
  --evm-address=<your metamask address>
```

## Security
To protect you keys please make sure you follow basic security rules

### Set up ssh keys for authentication
Good tutorial on how to set up ssh keys for authentication to your server can be found [here](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)

### Get list of validators
```
celestia-appd q staking validators -oj --limit=3000 | jq '.validators[] | select(.status=="BOND_STATUS_BONDED")' | jq -r '(.tokens|tonumber/pow(10; 6)|floor|tostring) + " \t " + .description.moniker' | sort -gr | nl
```

## Get currently connected peer list with ids
```
curl -sS http://localhost:26657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr)"' | awk -F ':' '{print $1":"$(NF)}'
```

## Usefull commands
### Service management
Check logs
```
journalctl -u celestia-appd -f -o cat
```

Start service
```
sudo systemctl start celestia-appd
```

Stop service
```
sudo systemctl stop celestia-appd
```

Restart service
```
sudo systemctl restart celestia-appd
```

### Node info
Synchronization info
```
celestia-appd status 2>&1 | jq .SyncInfo
```

Validator info
```
celestia-appd status 2>&1 | jq .ValidatorInfo
```

Node info
```
celestia-appd status 2>&1 | jq .NodeInfo
```

Show node id
```
celestia-appd tendermint show-node-id
```

### Wallet operations
List of wallets
```
celestia-appd keys list
```

Recover wallet
```
celestia-appd keys add $WALLET --recover
```

Delete wallet
```
celestia-appd keys delete $WALLET
```

Get wallet balance
```
celestia-appd query bank balances $CELESTIA_WALLET_ADDRESS
```

Transfer funds
```
celestia-appd tx bank send $CELESTIA_WALLET_ADDRESS <TO_CELESTIA_WALLET_ADDRESS> <COST>utia --gas 250000 --fees 25000utia
```

### Voting
```
celestia-appd tx gov vote 1 yes --from $WALLET --chain-id=$CELESTIA_CHAIN_ID --gas 250000 --fees 25000utia
```

### Staking, Delegation and Rewards
Delegate stake
```
celestia-appd tx staking delegate $CELESTIA_VALOPER_ADDRESS 1000000utia --from=$WALLET --chain-id=$CELESTIA_CHAIN_ID --gas=250000 --fees 25000utia
```

Redelegate stake from validator to another validator
```
celestia-appd tx staking redelegate <srcValidatorAddress> <destValidatorAddress> 1000000utia --from=$WALLET --chain-id=$CELESTIA_CHAIN_ID --gas=250000 --fees=25000utia
```

Withdraw all rewards
```
celestia-appd tx distribution withdraw-all-rewards --from=$WALLET --chain-id=$CELESTIA_CHAIN_ID --gas=250000 --fees=25000utia
```

Withdraw rewards with commision
```
celestia-appd tx distribution withdraw-rewards $CELESTIA_VALOPER_ADDRESS --from=$WALLET --commission --chain-id=$CELESTIA_CHAIN_ID --gas=250000 --fees=25000utia
```

### Validator management
Edit validator
```
celestia-appd tx staking edit-validator \
  --moniker=$NODENAME \
  --identity=<your_keybase_id> \
  --website="<your_website>" \
  --details="<your_validator_description>" \
  --chain-id=$CELESTIA_CHAIN_ID \
  --from=$WALLET \
  --gas=250000 \
  --fees=25000utia \
  --evm-address=<your metamask address>
```

Unjail validator
```
celestia-appd tx slashing unjail \
  --from=$WALLET \
  --chain-id=$CELESTIA_CHAIN_ID \
  --gas=250000 \
  --fees=25000utia
```

### Delete node
This commands will completely remove node from server. Use at your own risk!
```
sudo systemctl stop celestia-appd
sudo systemctl disable celestia-appd
sudo rm /etc/systemd/system/celestia-app* -rf
sudo rm $(which celestia-appd) -rf
sudo rm $HOME/.celestia-app* -rf
sudo rm $HOME/celestia-app -rf
sed -i '/CELESTIA_/d' ~/.bash_profile
```
