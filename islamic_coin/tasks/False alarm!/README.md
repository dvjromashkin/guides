A small instruction on how to put a notification system in telegrams, about stopping the synchronization of your node, as well as its automatic restart.

The guide is designed so that you know what a telegram bot is, and where and how to get a Telegram bot token and Chat-ID.

We go to our node, and execute a few commands. I execute them as root, but of course I do not advise you to do this.

``` 
nano /home/check.sh
```

Paste the following text, and replace Telegram bot token and chat-id

```

#!/bin/bash
source .bash_profile
source .profile
source .bashrc
RESULT=$(curl http://127.0.0.1:26657/status | grep "latest_block_height" | awk '{print $2}' | tr -d '",')
sleep 30
RESULT2=$(curl http://127.0.0.1:26657/status | grep "latest_block_height" | awk '{print $2}' | tr -d '",')
CHECKS=$(( $RESULT2 - $RESULT ))
if (( $RESULT > 0 ));
then
        echo -e "Syncing\n"
        echo "Block 1 =" $RESULT " Block 2 =" $RESULT2 " Difference =" $RESULT
else
TELEGRAM_BOT_TOKEN="0000000000:xXxXxXxXxXxXxXxXxXxX"
CHAT_ID="12341234123"

SUMMARY=$(echo YOUR NODE HAQQ STOPPED SYNCING!!! )

curl -X POST \
     -H "Content-Type: application/json" \
     -d "{\"chat_id\": \"${CHAT_ID}\", \"text\": \"${SUMMARY}\", \"disable_notification\": true}" \
     https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage
        sudo systemctl restart haqqd
        echo -e "Not syncing\n"
fi

```

Next, you need to save. You can do this in Nano with the keyboard shortcut (ctrl+s , and ctrl+x)

Further, we need to give our script permission to execute the following command.

```
chmod a+x /home/check.sh
```

Then, that it worked automatically, it needs to be registered in cron. To do this, open the crontab with the following command:

```
crontab -e
```
And paste the following line at the very bottom:

```
*/10 * * * * /home/check.sh >> /home/cron.log
```
Save and exit.

Fabulous. Now everything works offline, and sends you notifications that the node is out of sync.
