I want to provide you with a small guide for setting up a script to determine the status of the validator (jailed), automatically exit the jail, as well as notifications in Telegram

This guide is written as a continuation of the guide https://github.com/dvjromashkin/testnets-tasks/tree/main/islamic_coin/tasks/False%20alarm!

It also relies on the fact that you know where to get the Telegram bot token and chat ID.

Since many have a wallet password inside the node, the script will consist of two parts. The first part will call the main script and automatically enter the wallet password using "Expect".

Need to install "expect" first
```
sudo apt install expect -y
```
We create the first script that will run the main one and enter the wallet password:
```
nano /home/autounjail.sh
```
We insert the following text, where we replace the numbers 12345678 with your password:
```
#!/usr/bin/expect -f
set timeout -1

spawn ./unjail.sh
expect {
{Enter keyring passphrase:}
{send "12345678\r";exp_continue}
}
```
Save and exit. (ctrl+s, ctrl+x)
