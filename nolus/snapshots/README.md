# SnapShot (~2.1 GB) updated every 12 hours
```python
cd $HOME
sudo apt install lz4 -y
sudo systemctl stop nolusd
cp $HOME/.nolus/data/priv_validator_state.json $HOME/.nolus/priv_validator_state.json.backup
rm -rf $HOME/.nolus/data
curl -L https://snapshot-nolus.cryptech.com.ua/nolus-snapshot.tar.lz4 | tar -Ilz4 -xf - -C $HOME/
mv $HOME/.nolus/priv_validator_state.json.backup $HOME/.nolus/data/priv_validator_state.json
sudo systemctl restart nolusd && journalctl -u nolusd -f -o cat
```
