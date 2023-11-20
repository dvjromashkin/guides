<p style="font-size:14px" align="center">
<a href="https://cryptech.com.ua/" target="_blank">Visit our website <img src="https://storage.googleapis.com/c-pictures/logo_6c33163af2_dbef49e129_d981e5eee8/logo_6c33163af2_dbef49e129_d981e5eee8.png" width="150"/></a>
</p>

<h1 align="center"> Source Protocol Node/Validator guide</h1>

>- [WebSite](https://www.sourceprotocol.io/) 
>- [GitHub](https://github.com/Source-Protocol-Cosmos/)
>- [OFFICIAL DOCS](https://docs.sourceprotocol.io/) 
>- [OUR EXPLORER](https://explorers.cryptech.com.ua/sourcechain-testnet) 

- **Recommended hardware requirements**:

| Node Type |CPU | RAM  | Storage  | 
|-----------|----|------|----------|
| Testnet   |   4|  8GB | 200GB    |

# Our services:

RPC
>- https://rpc-source.cryptech.com.ua

API
>- https://api-source.cryptech.com.ua

GRPC
>- https://grpc-source.cryptech.com.ua

# Installation

### Install dependencies

```python
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

## Install GO 1.19
```python
ver="1.19" && \
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz" && \
sudo rm -rf /usr/local/go && \
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz" && \
rm "go$ver.linux-amd64.tar.gz" && \
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile && \
source $HOME/.bash_profile && \
go version
```

# Build bin
```python
cd $HOME
git clone https://github.com/Source-Protocol-Cosmos/source.git --branch testnet
cd source
make install
```


```python
sourced init <nodename> --chain-id sourcechain-testnet
sourced config chain-id sourcechain-testnet
```    

## Create/recover wallet
```python
sourced keys add <walletname>
  OR
sourced keys add <walletname> --recover
```

## Download Genesis
```python
wget -qO $HOME/.source/config/genesis.json "https://raw.githubusercontent.com/Source-Protocol-Cosmos/testnets/master/sourcechain-testnet/genesis.json"
```

## Set up the minimum gas price 
```python
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0usource\"/" $HOME/.source/config/app.toml
```

### Pruning (optional)
```python
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.source/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.source/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.source/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.source/config/app.toml
```
### Indexer (optional) 
```python
indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.sourced/config/config.toml
```

## StateSync
```python
SNAP_RPC=https://rpc-source.cryptech.com.ua:443/
peers="9d16b552697cdce3c8b4f23de53708533d99bc59@165.232.144.133:26656,d565dd0cb92fa4b830662eb8babe1dcdc340c321@44.234.26.62:26656,2dbc3e6d52e5eb9357aec5cf493718f6078ffaad@144.76.224.246:36656,330b14f94d3bbe6c4059f31bd8fbf9960cf1387e@185.144.99.3:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.source/config/config.toml
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 100)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.source/config/config.toml
c4ed tendermint unsafe-reset-all --home /root/.sourced --keep-addr-book
sudo systemctl restart sourced && journalctl -u sourced -f -o cat
```

# Create a service file
```python
sudo tee /etc/systemd/system/sourced.service > /dev/null <<EOF
[Unit]
Description=source-node
After=network-online.target

[Service]
User=$USER
ExecStart=$(which sourced) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Start
```python
sudo systemctl daemon-reload
sudo systemctl enable sourced
sudo systemctl restart sourced && sudo journalctl -u sourced -f -o cat
```

### Create validator
```python
sourced tx staking create-validator \
  --amount 1000000source \
  --from <walletName> \
  --commission-max-change-rate "0.1" \
  --commission-max-rate "0.2" \
  --commission-rate "0.1" \
  --min-self-delegation "1" \
  --pubkey  $(sourced tendermint show-validator) \
  --moniker <your moniker> \
  --chain-id sourcechain-testnet \
  --identity="" \
  --details="" \
  --website="" -y
```

## Delete node
```python
sudo systemctl stop sourced && \
sudo systemctl disable sourced && \
rm /etc/systemd/system/sourced.service && \
sudo systemctl daemon-reload && \
cd $HOME && \
rm -rf source && \
rm -rf .source && \
rm -rf $(which sourced)
```
