# Setup Bridge Node Guide

## Hardware Requirements
Like any Cosmos-SDK chain, the hardware requirements are pretty modest.

Mandatory requirement for installation - GO ver =<1.20

Stabe RPC connect (recommendated local rpc)

### Hardware Requirements
 - Memory: 8 GB RAM
 - CPU: 6 cores
 - Disk: 500 GB SSD Storage
 - Bandwidth: 1 Gbps for Download/1 Gbps for Upload


## Download and build binaries

```
cd $HOME 
rm -rf celestia-node 
git clone https://github.com/celestiaorg/celestia-node.git 
cd celestia-node
git checkout v0.10.4
make build
sudo mv build/celestia /usr/local/bin
make cel-key
sudo mv cel-key /usr/local/bin
```

## Add Bridge wallet

### GENERATE NEW WALLET

```
cel-key add bridge-wallet --node.type bridge --p2p.network mocha-3
```

### RECOVER EXISTING WALLET

```
cel-key add bridge-wallet --node.type bridge --p2p.network mocha-3 --recover
```

## Fund the wallet with testnet tokens

Once you start the Bridge Node, a wallet key will be generated for you. You will need to fund that address with Testnet tokens to pay for PayForBlob transactions

## Initialize Bridge node

```
celestia bridge init \
  --keyring.accname bridge-wallet \
  --core.ip http://<your-rpc-node> \
  --core.rpc.port 26657 \
  --core.grpc.port 9090 \
  --p2p.network mocha-3 
```

## Create Service

```
sudo tee /etc/systemd/system/celestia-bridge.service > /dev/null << EOF
[Unit]
Description=Celestia Bridge Node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which celestia) bridge start \\
--keyring.accname bridge-wallet \\
--core.ip http://<your-rpc-node> \\
--core.rpc.port 26657 \\
--core.grpc.port 9090 \\
--p2p.network mocha-3 \\
Restart=on-failure
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable celestia-bridge
```

## Start Bridge node

```
systemctl restart celestia-bridge
```

## Check Bridge node logs

```
journalctl -fu celestia-bridge -o cat
```
