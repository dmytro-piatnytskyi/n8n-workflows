# Shopping List Bot

A Telegram bot for a shared family shopping list, built in n8n.

## What it does
- Any message adds an item to the list and saves who added it
- "список" shows the full list with names
- "очистити" clears the list

## How it works
Telegram Trigger receives a message, Switch checks the command, n8n Data Tables store the list, and the bot replies in the same chat.

## Tools
n8n, Telegram Bot API, n8n Data Tables, JavaScript (Code node)
