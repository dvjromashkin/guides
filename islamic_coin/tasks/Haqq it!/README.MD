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

Next, we need to create and configure the actual script for automatically checking and logging out of the jail:
```
nano /home/unjail.sh
```
We insert the following text, and replace some data in it with our own:
<your valoper key>
<your wallet name>
Telegram bot ID
Chat ID
```
#!/bin/bash
source /root/.bashrc
source /root/.profile
source /root/.bash_profile
STATUS=$(haqqd q staking validator <you valoper key> | grep jailed | awk '{print $2}')
if [[ $STATUS == false ]];
then
        echo -e "Not jailed\n"
else
TELEGRAM_BOT_TOKEN="1234567890:xXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxXxX"
CHAT_ID="123456789"

SUMMARY=$(echo HAQQ: In Jail. Unjailed is run. )

curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"chat_id\": \"${CHAT_ID}\", \"text\": \"${SUMMARY}\", \"disable_notification\": true}" \
     https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage
        haqqd tx slashing unjail --from <your wallet name> --gas auto --chain-id haqq_54211-3 -y
        echo -e "In jailed\n"
fi

```
  
Save and exit (ctrl+s, ctrl+x)

  Next, we need to allow the scripts our right to run.
  
```
chmod +x /home/autounjail.sh
chmod +x /home/unjail.sh
```
  
Now we register it in crowns, for every 10 minutes, and enjoy.
  
```
crontab -e 
```
and paste at the bottom
```
*/10 * * * * /root/autounjail.sh >> /home/cron.log
```
  Save and exit (ctrl+s, ctrl+x)
