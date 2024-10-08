<p style="font-size:14px" align="center">
<a href="https://cryptech.com.ua/" target="_blank">Visit our website <img src="https://storage.googleapis.com/c-pictures/logo_6c33163af2_dbef49e129_d981e5eee8/logo_6c33163af2_dbef49e129_d981e5eee8.png" width="150"/></a>
</p>

<h1 align="center"> C4E MAINNET guide</h1>

>- [WebSite](https://c4e.io/) 
>- [GitHub](https://github.com/chain4energy)
>- [EXPLORER](https://explorer.c4e.io/) 

- **Minimum hardware requirements**:

| Node Type |CPU | RAM  | Storage  | 
|-----------|----|------|----------|
| Mainnet   |   8|  16GB | 300GB    |

# Our services:

RPC
>- https://rpc-c4e.cryptech.com.ua

API
>- https://api-c4e.cryptech.com.ua

GRPC
>- https://grpc-c4e.cryptech.com.ua

# Installation

### Install dependencies

```python
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```

## Install GO 1.19
```python
ver="1.20.6" && \
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
git clone https://github.com/chain4energy/c4e-chain
cd c4e-chain
git checkout v1.3.1
make install
```
`c4ed version --long`
- version: v1.3.1
- commit: 55fe5c3c1a1b9a8489d39e765413315c5b61c4f7

```python
c4ed init <nodename> --chain-id perun-1
c4ed config chain-id perun-1
```    

## Create/recover wallet
```python
c4ed keys add <walletname>
  OR
c4ed keys add <walletname> --recover
```

## Download Genesis
```python
wget https://github.com/chain4energy/c4e-chains/raw/main/perun-1/genesis.json -O $HOME/.c4e-chain/config/genesis.json
```

## Set up the minimum gas price 
```python
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0uc4e\"/" $HOME/.c4e-chain/config/app.toml
```

### Pruning (optional)
```python
pruning="custom" && \
pruning_keep_recent="100" && \
pruning_keep_every="0" && \
pruning_interval="10" && \
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" ~/.c4e-chain/config/app.toml && \
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" ~/.c4e-chain/config/app.toml && \
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" ~/.c4e-chain/config/app.toml && \
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" ~/.c4e-chain/config/app.toml
```
### Indexer (optional) 
```python
indexer="null" && \
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.c4e-chain/config/config.toml
```

## Download addrbook
```python
wget -O $HOME/.c4e-chain/config/addrbook.json "https://github.com/dvjromashkin/testnets-tasks/raw/main/c4e-main/addrbook.json"
```
## StateSync
```python
SNAP_RPC=https://rpc-c4e.cryptech.com.ua:443/
peers="15ea07bf6211f708eb2966b6c66e3aaa45834137@185.144.99.37:26656"
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$peers\"/" $HOME/.c4e-chain/config/config.toml
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 100)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)

echo $LATEST_HEIGHT $BLOCK_HEIGHT $TRUST_HASH

sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.c4e-chain/config/config.toml
c4ed tendermint unsafe-reset-all --home /root/.c4e-chain --keep-addr-book
sudo systemctl restart c4ed && journalctl -u c4ed -f -o cat
```
## SnapShot (~0.1 GB) updated every 12 hours
```python
cd $HOME
apt install lz4
sudo systemctl stop c4ed
cp $HOME/.c4e-chain/data/priv_validator_state.json $HOME/.c4e-chain/priv_validator_state.json.backup
rm -rf $HOME/.c4e-chain/data
curl -o - -L http://c4e.snapshot.stavr.tech:1018/c4e/c4e-snap.tar.lz4 | lz4 -c -d - | tar -x -C $HOME/.c4e-chain --strip-components 2
mv $HOME/.c4e-chain/priv_validator_state.json.backup $HOME/.c4e-chain/data/priv_validator_state.json
wget -O $HOME/.c4e-chain/config/addrbook.json "https://raw.githubusercontent.com/obajay/nodes-Guides/main/Projects/C4E/addrbook.json"
sudo systemctl restart c4ed && journalctl -u c4ed -f -o cat
```

# Create a service file
```python
sudo tee /etc/systemd/system/c4ed.service > /dev/null <<EOF
[Unit]
Description=c4e
After=network-online.target

[Service]
User=$USER
ExecStart=$(which c4ed) start
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
sudo systemctl enable c4ed
sudo systemctl restart c4ed && sudo journalctl -u c4ed -f -o cat
```

### Create validator
```python
c4ed tx staking create-validator \
  --amount 1000000uc4e \
  --from <walletName> \
  --commission-max-change-rate "0.1" \
  --commission-max-rate "0.2" \
  --commission-rate "0.1" \
  --min-self-delegation "1" \
  --pubkey  $(c4ed tendermint show-validator) \
  --moniker <yourValidatorMonioker> \
  --chain-id perun-1 \
  --identity="" \
  --details="" \
  --website="" -y
```

## Delete node
```python
sudo systemctl stop c4ed && \
sudo systemctl disable c4ed && \
rm /etc/systemd/system/c4ed.service && \
sudo systemctl daemon-reload && \
cd $HOME && \
rm -rf c4e-chain && \
rm -rf .c4e-chain && \
rm -rf $(which c4ed)
```
