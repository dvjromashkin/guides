![Penumbra logo](docs/images/penumbra-dark.svg#gh-dark-mode-only)
![Penumbra logo](docs/images/penumbra-light-bw.svg#gh-light-mode-only)


So I want to tell you how to run a validator on the Penumbra network. The problem is that the network is constantly being updated (about every week) and needs to be reinstalled each time. Which complicates the process of building a one-line script.
First, let’s try it manually and figure out how it works.
Penumbra is a mixture of two applications. One represents the mechanics itself, and is written in Rust, and the other is Tendermint, which is needed to work the blockchain structure itself.

Preparations
----------------------------------------------------------------------------------------------
To install we need the latest Go and Rust
```
sudo apt-get install build-essential pkg-config libssl-dev clang -y

cd $HOME

sudo apt update

wget -O go1.18.4.linux-amd64.tar.gz https://golang.org/dl/go1.18.4.linux-amd64.tar.gz

rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.4.linux-amd64.tar.gz && rm go1.18.4.linux-amd64.tar.gz

echo 'export GOROOT=/usr/local/go' >> $HOME/.bash_profile
echo 'export GOPATH=$HOME/go' >> $HOME/.bash_profile
echo 'export GO111MODULE=on' >> $HOME/.bash_profile
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.bash_profile && . $HOME/.bash_profile

go version

sudo apt update

sudo curl https://sh.rustup.rs -sSf | sh -s -- -y

. $HOME/.cargo/env
```

Tendermint installation
------------------------------------------------------------------------------------------
Next, we need to install Tendermint v0.34.21
```
git clone https://github.com/tendermint/tendermint.git

git checkout v0.34.21

make install

tendermint version

cd $HOME
```
Next, we need to create a service file for Tendermint to work.
```
echo "[Unit]
Description=Tendermint Node
After=network.target
[Service]
User=$USER
Type=simple
ExecStart=$(which tendermint) start --home $HOME/.penumbra/testnet_data/node0/tendermint
Restart=on-failure
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target" > $HOME/tendermintd.service
sudo mv $HOME/tendermintd.service /etc/systemd/system
sudo tee <<EOF >/dev/null /etc/systemd/journald.conf
Storage=persistent
EOF

sudo systemctl restart systemd-journald

sudo systemctl daemon-reload

sudo systemctl enable tendermintd
```
Installing and Configuring Penumbra
------------------------------------------------------------------------------------------------
Next, you need to install Penumbra itself by copying two files: pd, pcli. And also send them to /usr/local/bin
```
git clone https://github.com/penumbra-zone/penumbra

cd penumbra/

cargo build --release --bin pd && cargo build --release --bin pcli

cp $HOME/penumbra/target/release/pd /usr/local/bin
cp $HOME/penumbra/target/release/pcli /usr/local/bin
```
Next, we need to create new keys with the command:
```
pcli keys generate
```
In the output, you will get information about the path of the backup file, and the mnemonic phrase, which you should definitely write down. Or restore them by importing the mnemonic phrase:
```
pcli keys import phrase <your mnemonic phrase>
```
You can find out your address by using the command:
```
pcli v address
```
Using this address, you need to request tokens from the faucet. To do this, join the discord channel at the link https://discord.gg/hKvkrqa3zC (if the link does not work, you can find it out from their website: https://penumbra.zone/ under “Community”), and go to #testnet-faucet and put in your Penumbra address.

The balance can be checked with a command:
```
pcli v balance
```
Don’t be intimidated by the long process. Wallet synchronizes with the network.
Now, when the balance is there, we can start to run the node, and create a validator.
Configure, and prepare Tendermint to work.
```
pd testnet join
```
Next we need to set the name of our validator, and do a few manipulations with Tendermint. Specify the name of your validator instead of <MONIKER>
```
VALIDATOR=<MONIKER>
sed -i.bak "s/^moniker *=.*/moniker = \"$VALIDATOR\"/;" $HOME//.penumbra/testnet_data/node0/tendermint/config/config.toml
```
  Next, you need to create a JSON validator file. The next command creates a template file.

```
pcli validator definition template --file $HOME/.penumbra/validator.json
```
  And edit it:
```
PVKEY=$(cat $HOME/.penumbra/testnet_data/node0/tendermint/config/priv_validator_key.json | jq ".pub_key.value")
sed -i.bak "3c\  \"consensus_key\": ${PVKEY}," $HOME/.penumbra/validator.json
sed -i.bak "4c\  \"name\": \"${VALIDATOR}\"," $HOME/.penumbra/validator.json
sed -i.bak "7c\  \"enabled\": true," $HOME/.penumbra/validator.json
```
  Now that almost everything is ready, we create a service for the penumbra:
```
echo "[Unit]
Description=Penumbra Node
After=network.target
[Service]
User=$USER
Type=simple
ExecStart=$(which pd) start --home $HOME/.penumbra/testnet_data/node0/pd
Restart=on-failure
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target" > $HOME/pdd.service

sudo mv $HOME/pdd.service /etc/systemd/system

sudo tee <<EOF >/dev/null /etc/systemd/journald.conf
Storage=persistent
EOF

sudo systemctl restart systemd-journald

sudo systemctl daemon-reload

sudo systemctl enable pdd
  ```
Next, you need to run both services in a strictly defined order. First we start the PD
```
systemctl start pdd
```
  and then Tendermint
```
systemctl start tendermintd
```
  It happens that a node does not start right away. This is due to the fact that the main peer is overloaded, and you need to add possible peers.
If you need to restart the node, you need to clear both databases completely:
```
systemctl stop tendermintd

systemctl stop pdd

rm -rf /root/.penumbra/testnet_data/node0/pd

tendermint unsafe-reset-all --home=/root/.penumbra/testnet_data/node0/tendermint
```
  Then the alternating starts.
```
systemctl start pdd
systemctl start tendermintd
```
  Checking the logs:
```
journalctl -u tendermintd -f -o cat
  ```
Then we wait for our node to synchronize. You can check the status with the following command:
```
curl -sS http://127.0.0.1:26657/status | jq -r '.result.sync_info.catching_up'
  ```
If “true” — the node is still synchronized. If “false” has already finished synchronization.

Forming the validator and the steak
-----------------------------------------------------------------------------------
If our node is already synchronized, we need to make our validator data sent to the chain.
To begin with, we can look at the list of available validators. To make it work better, we will do it from under our RPC, with the following command
```
pcli --node 127.0.0.1 --tendermint-port 26657 q validator list
```
  We reset the wallet from the previous network:
```
pcli --node 127.0.0.1 --tendermint-port 26657 v reset
```
  Next, we check our balance:
```
pcli --node 127.0.0.1 --tendermint-port 26657 v balance
```
  Upload our validator file to the network:
```
pcli validator definition upload --file /root/.penumbra/validator.json
```
  We check its inactive status with the command:
```
pcli -n 127.0.0.1 --tendermint-port 26657 q validator list -i
```
  The address of our validator can be seen with the command:
```
cat /root/.penumbra/validator.json | jq ".identity_key" | tr -d '"'
```
  We need to delegate coins to our validator. To delegate everything, you can use the command:
```
pcli --node 127.0.0.1 --tendermint-port 26657 tx delegate $(pcli --node 127.0.0.1 --tendermint-port 26657 v balance | grep penumbra) --to $(cat /root/.penumbra/validator.json | jq ".identity_key" | tr -d '"')
```
  The status of the active validator, will not appear immediately. Check periodically:
```
pcli --node 127.0.0.1 --tendermint-port 26657 q validator list
```
  Updating
  -------------------------------------------------------------------------------------------------
Since the renewal of each network, implies a complete reset of the network, and the recharge of the balance on the purse, which you, at least once, shone in the faucet. Then the procedure for resetting should be done almost completely.
We assume that you already have the validator.json file in /root/.penumbra/ and therefore we need to reinstall the client itself.
```
systemctl stop tendermintd

systemctl stop pdd

rm -rf $HOME/penumbra/
  
rm -rf $HOME/.penumbra/testnet_data/node0/pd

git clone https://github.com/penumbra-zone/penumbra

cd penumbra/

git pull

cargo build --release --bin pd && cargo build --release --bin pcli

cp $HOME/penumbra/target/release/pd /usr/local/bin

cp $HOME/penumbra/target/release/pcli /usr/local/bin

rm -rf $HOME/.penumbra/testnet_data/node0/pd

tendermint unsafe_reset_all --home="$HOME/.penumbra/testnet_data/node0/tendermint"

curl -s http://testnet.penumbra.zone:26657/genesis | jq ".result.genesis" > $HOME/.penumbra/testnet_data/node0/tendermint/config/genesis.json

IDNODE=$(curl -s http://testnet.penumbra.zone:26657/status | jq ".result.node_info.id" | tr -d '"')

PEER=$IDNODE@testnet.penumbra.zone:26656

sed -i.bak "s/^persistent-peers *=.*/persistent-peers = \"$PEER\"/;" $HOME/.penumbra/testnet_data/node0/tendermint/config/config.toml

sed -i.bak "s/^bootstrap-peers *=.*/bootstrap-peers = \"$PEER\"/;" $HOME/.penumbra/testnet_data/node0/tendermint/config/config.toml

sed -i.bak "s/^seeds *=.*/seeds = \"$PEER\"/;" $HOME/.penumbra/testnet_data/node0/tendermint/config/config.toml

systemctl start pdd

systemctl start tendermintd
```
  Checking the logs:
```
journalctl -u tendermintd -f -o cat
```
  Then we wait for our node to synchronize. You can check the status with the following command:
```
curl -sS http://127.0.0.1:26657/status | jq -r '.result.sync_info.catching_up'
```
  If “true” — the node is still synchronized. If “false” has already finished synchronization, you can proceed to the formation of the Validator and the steak specified above in the instructions.

Problems with Peers
--------------------------------------------------------------------------------------------
```
PEERS=$(curl -sS http://testnet.penumbra.zone:26657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):\(.node_info.listen_addr),"' | awk -F ':' '{print $1":"$(NF)}' | tr -d '\n' | sed 's/.$//')
sed -i.bak "s/^persistent-peers *=.*/persistent-peers = \"$PEERS\"/;" $HOME/.penumbra/testnet_data/node0/tendermint/config/config.toml
```
