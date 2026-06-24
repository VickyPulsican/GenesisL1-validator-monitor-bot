<p align="center">
  <img src="genesisl1-logo.png" width="180">
</p>

<h1 align="center">GenesisL1 Validator Monitor Bot</h1>

<p align="center">
A lightweight Telegram monitoring bot for GenesisL1 validators.
</p>

# GenesisL1 Validator Monitor Bot

A lightweight Telegram monitoring bot for GenesisL1 validators.

This bot sends regular status updates to Telegram and helps validators monitor node health, validator status, staking information, block sync, system resources, and live L1 price.

## Example Output

![GenesisL1 Validator Monitor](Bot%20Image.png)

## Features

* Node process status
* Sync status
* Validator active set status
* Voting power
* Self delegation
* Self stake value in USD
* Live L1 price
* Chain block height
* Local node block height
* Blocks behind
* RAM usage
* CPU load
* Root disk usage
* Data disk usage
* VPS uptime
* Genesis service uptime
* Telegram notifications every 5 minutes

## Requirements

* Ubuntu 22.04 or 24.04
* GenesisL1 node already installed and synced
* `genesisd` running as a systemd service
* Local RPC enabled
* Telegram bot token
* Telegram chat ID
* Wallet address
* Validator valoper address

## Install Dependencies

```bash
sudo apt update
sudo apt install curl jq cron -y
```

Verify:

```bash
curl --version
jq --version
```

## Telegram Bot Setup

Create a bot using Telegram BotFather.

Open Telegram and search:

```text
@BotFather
```

Run:

```text
/newbot
```

Save the bot token.

Then send `/start` to your new bot.

Get your Telegram chat ID:

```bash
curl -s "https://api.telegram.org/botYOUR_BOT_TOKEN/getUpdates"
```

Find:

```json
"chat":{"id":YOUR_CHAT_ID}
```

## Required Values

Before creating the script, prepare these values:

```bash
BOT_TOKEN="YOUR_BOT_TOKEN"
CHAT_ID="YOUR_CHAT_ID"
WALLET_ADDRESS="YOUR_WALLET_ADDRESS"
VALOPER_ADDRESS="YOUR_VALOPER_ADDRESS"
```

Wallet address example:

```text
genesis1xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Validator address example:

```text
genesisvaloper1xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Create Monitor Script

Create the script:

```bash
nano /root/genesis_monitor.sh
```

Paste this script:

```bash
#!/bin/bash

BOT_TOKEN="YOUR_BOT_TOKEN"
CHAT_ID="YOUR_CHAT_ID"

LOCAL_RPC="http://127.0.0.1:36657"
PUBLIC_RPC="https://genesisl1.rpc.utsa.tech"
SERVICE="genesisd"

WALLET_ADDRESS="YOUR_WALLET_ADDRESS"
VALOPER_ADDRESS="YOUR_VALOPER_ADDRESS"

LOCAL_HEIGHT=$(curl -s --max-time 10 "$LOCAL_RPC/status" | jq -r '.result.sync_info.latest_block_height // "0"')
PUBLIC_HEIGHT=$(curl -s --max-time 10 "$PUBLIC_RPC/status" | jq -r '.result.sync_info.latest_block_height // "0"')
CATCHING_UP=$(curl -s --max-time 10 "$LOCAL_RPC/status" | jq -r '.result.sync_info.catching_up' | tr -d '[:space:]')
NODE_TIME=$(curl -s --max-time 10 "$LOCAL_RPC/status" | jq -r '.result.sync_info.latest_block_time // "unknown"')

if [[ "$LOCAL_HEIGHT" =~ ^[0-9]+$ ]] && [[ "$PUBLIC_HEIGHT" =~ ^[0-9]+$ ]]; then
  BEHIND=$((PUBLIC_HEIGHT - LOCAL_HEIGHT))
else
  BEHIND="unknown"
fi

if systemctl is-active --quiet "$SERVICE"; then
  SERVICE_STATUS="✅ Running"
else
  SERVICE_STATUS="❌ Not Running"
fi

if [ "$CATCHING_UP" = "false" ]; then
  SYNC_STATUS="✅ Fully Synced"
elif [ "$CATCHING_UP" = "true" ]; then
  SYNC_STATUS="⏳ Syncing"
else
  SYNC_STATUS="⚠️ Unknown"
fi

VALIDATOR_STATUS=$(/root/go/bin/genesisd query staking validator "$VALOPER_ADDRESS" --node tcp://127.0.0.1:36657 --output json 2>/dev/null | jq -r '.validator.status')

if [ "$VALIDATOR_STATUS" = "BOND_STATUS_BONDED" ]; then
  VALIDATOR_DISPLAY="✅ Active (Validator Set)"
else
  VALIDATOR_DISPLAY="⚠️ ${VALIDATOR_STATUS:-Unknown}"
fi

VOTING_POWER_RAW=$(/root/go/bin/genesisd query staking validator "$VALOPER_ADDRESS" --node tcp://127.0.0.1:36657 --output json 2>/dev/null | jq -r '.validator.tokens')
VOTING_POWER=$(awk "BEGIN {printf \"%'0.2f\", $VOTING_POWER_RAW/1000000000000000000}")

SELF_DELEGATION_RAW=$(/root/go/bin/genesisd query staking delegation "$WALLET_ADDRESS" "$VALOPER_ADDRESS" --node tcp://127.0.0.1:36657 --output json 2>/dev/null | jq -r '.delegation_response.balance.amount')
SELF_DELEGATION=$(awk "BEGIN {printf \"%.0f\", $SELF_DELEGATION_RAW/1000000000000000000}")

L1_PRICE=$(curl -s --max-time 10 http://46.224.42.12:8585/price.txt | awk '{print $2}')
L1_PRICE_NUM=$(echo "$L1_PRICE" | tr -d '$')
SELF_STAKE_VALUE=$(awk "BEGIN {printf \"\$%.2f\", $SELF_DELEGATION * $L1_PRICE_NUM}")

CPU_LOAD=$(uptime | awk -F'load average:' '{print $2}' | xargs)
RAM_USAGE=$(free -h | awk '/Mem:/ {printf "%s/%s (%.1f%%)", $3, $2, ($3/$2)*100}')
DISK_ROOT=$(df -h / | awk 'NR==2 {print $3 "/" $2 " (" $5 ")"}')
DISK_DATA=$(df -h /data 2>/dev/null | awk 'NR==2 {print $3 "/" $2 " (" $5 ")"}')
UPTIME=$(uptime -p)

GENESIS_UPTIME_SECONDS=$(systemctl show genesisd --property=ActiveEnterTimestampMonotonic --value)
GENESIS_UPTIME_SECONDS=$(( ($(cut -d' ' -f1 /proc/uptime | cut -d'.' -f1) * 1000000 - GENESIS_UPTIME_SECONDS) / 1000000 ))
GENESIS_UPTIME="$(($GENESIS_UPTIME_SECONDS/86400))d $((($GENESIS_UPTIME_SECONDS%86400)/3600))h"

MESSAGE="📊 GenesisL1 Status Update

🟢 NODE & VALIDATOR
━━━━━━━━━━━━━━━━━━━━
Node Process:      $SERVICE_STATUS
Sync Status:       $SYNC_STATUS
Validator:         $VALIDATOR_DISPLAY

🏛️ STAKING
━━━━━━━━━━━━━━━━━━━━
Voting Power:      $VOTING_POWER L1
Self Delegation:   $SELF_DELEGATION L1
Self Stake Value:  $SELF_STAKE_VALUE
L1 Price:          $L1_PRICE

⛓️ BLOCK INFO
━━━━━━━━━━━━━━━━━━━━
Chain Block:       $PUBLIC_HEIGHT
Node Block:        $LOCAL_HEIGHT
Blocks Behind:     $BEHIND

💻 SYSTEM RESOURCES
━━━━━━━━━━━━━━━━━━━━
RAM:               $RAM_USAGE
CPU Load:          $CPU_LOAD
Root Disk:         $DISK_ROOT
Data Disk:         ${DISK_DATA:-N/A}

⏰ TIME & UPTIME
━━━━━━━━━━━━━━━━━━━━
Last Block Time:   $NODE_TIME
VPS Uptime:        $UPTIME
Genesis Uptime:    $GENESIS_UPTIME

🤖 Built for GenesisL1 Validators"

curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
  -d chat_id="${CHAT_ID}" \
  --data-urlencode text="$MESSAGE" >/dev/null
```

Save:

```text
CTRL + O
Enter
CTRL + X
```

Make it executable:

```bash
chmod +x /root/genesis_monitor.sh
```

## Manual Test

Check syntax:

```bash
bash -n /root/genesis_monitor.sh
```

Run manually:

```bash
bash /root/genesis_monitor.sh
```

You should receive a Telegram message.

## Add Cron Job

Open crontab:

```bash
crontab -e
```

Add this line:

```bash
*/5 * * * * /bin/bash /root/genesis_monitor.sh >> /var/log/genesis_monitor.log 2>&1
```

Save and exit.

Verify cron:

```bash
crontab -l
```

Check logs:

```bash
tail -30 /var/log/genesis_monitor.log
```

## Useful Verification Commands

Check Genesis service:

```bash
systemctl status genesisd --no-pager
```

Check local RPC:

```bash
curl -s http://127.0.0.1:36657/status | jq '.result.sync_info'
```

Check public RPC:

```bash
curl -s https://genesisl1.rpc.utsa.tech/status | jq '.result.sync_info.latest_block_height'
```

Check validator status:

```bash
/root/go/bin/genesisd query staking validator YOUR_VALOPER_ADDRESS \
--node tcp://127.0.0.1:36657 \
--output json | jq '.validator.status'
```

Check self delegation:

```bash
/root/go/bin/genesisd query staking delegation \
YOUR_WALLET_ADDRESS \
YOUR_VALOPER_ADDRESS \
--node tcp://127.0.0.1:36657 \
--output json
```

## Example Output

```text
📊 GenesisL1 Status Update

🟢 NODE & VALIDATOR
━━━━━━━━━━━━━━━━━━━━
Node Process:      ✅ Running
Sync Status:       ✅ Fully Synced
Validator:         ✅ Active (Validator Set)

🏛️ STAKING
━━━━━━━━━━━━━━━━━━━━
Voting Power:      55,095.56 L1
Self Delegation:   83 L1
Self Stake Value:  $3.30
L1 Price:          $0.039805

⛓️ BLOCK INFO
━━━━━━━━━━━━━━━━━━━━
Chain Block:       13174500
Node Block:        13174500
Blocks Behind:     0
```

## Troubleshooting

### No Telegram Message

Check bot token:

```bash
curl -s "https://api.telegram.org/botYOUR_BOT_TOKEN/getMe"
```

Check chat ID:

```bash
curl -s "https://api.telegram.org/botYOUR_BOT_TOKEN/getUpdates"
```

### Script Has Errors

Run:

```bash
bash -n /root/genesis_monitor.sh
```

### Node Block Is Not Updating

Check local RPC:

```bash
curl -s http://127.0.0.1:36657/status | jq '.result.sync_info'
```

### Validator Status Shows Unknown

Make sure your valoper address is correct:

```bash
/root/go/bin/genesisd keys show YOUR_KEY_NAME --bech val -a
```

### L1 Price Not Showing

The script uses this community API:

```text
http://46.224.42.12:8585/price.txt
```

Test it manually:

```bash
curl -s --max-time 10 http://46.224.42.12:8585/price.txt
```

## Security Notes

Never publish:

* Telegram bot token
* Private keys
* Validator private key
* Node key
* Seed phrase
* Server passwords

Your wallet address and valoper address are public, but your bot token should always remain private.

## Credits

Built for GenesisL1 validators.

Created by **Vicky_Pulsican**
GenesisL1 Validator & Node Operator

Special thanks to **@lesnik13utsa** for the excellent validator guides, explorer infrastructure, and continuous support for GenesisL1 node operators, and to **@VanPez** for providing the open-source L1 Price API used in this project.

Additional thanks to the GenesisL1 community members who shared ideas, feedback, testing results, and suggestions that helped improve this monitoring bot.

