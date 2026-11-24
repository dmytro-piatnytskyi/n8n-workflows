# PartsPoint Auto: AI Sales Assistant for an Auto Parts Store

A Telegram AI assistant for a fictional online auto parts store in Kharkiv, Ukraine. It's built with self-hosted n8n and answers customers the way a good store employee would: it explains delivery and returns, finds parts that fit a specific car, checks order status, and hands tricky questions over to a human manager.

This is a portfolio demo. All store data is made up.

## What the bot can do

1. **Answer questions from a knowledge base (RAG).** Delivery, payment, returns, warranty, opening hours and FAQ. The knowledge base is a Google Doc that gets split into chunks and stored in a vector store.
2. **Find parts in the catalog.** It asks for the car make, model and year, then suggests only compatible parts from a Google Sheets catalog: name, brand, price, stock (or delivery time) and SKU.
3. **Show order status.** The customer gives an order number like `PP-24100`, and the bot shows the status and the Nova Poshta tracking number only after the last 4 digits of the phone number match.
4. **Hand off to a manager (human-in-the-loop).** Complex requests (VIN lookup, placing an order, discounts, complaints) go to a managers' Telegram group. A manager just hits "Reply", the bot forwards the answer to the customer and confirms in the group: "✅ Answer sent to the customer".
5. **Process returns.** The bot checks the order and the return rules (14 days, part not installed), collects the details one step at a time, adds a request to the sheet and notifies the managers.
6. **Send a daily AI report at 7 PM.** Number of chats and customers, top topics, what was escalated, what the bot couldn't answer, and suggestions for the knowledge base.

## How it works

The project is three n8n workflows. All of them use the `Europe/Kyiv` time zone.

### 1. `knowledge-base-loader.json`: Knowledge base loader

Run manually. It reads the Google Doc, splits it into chunks, creates embeddings with Google Gemini and saves them to an in-memory vector store (memory key `partspoint_kb`).

### 2. `ai-consultant.json`: AI consultant

One Telegram trigger with two branches (a bot can only have one trigger).

**Customer branch.** A filter lets through only private text messages, so the bot stays quiet in groups and ignores stickers and photos. Then an AI Agent replies, using:
- Google Gemini as the main model and a free OpenRouter model as a fallback;
- Simple Memory (last 10 messages per chat);
- 5 tools:

| Tool | What it does |
|---|---|
| `knowledge_base` | Searches the store rules by meaning |
| `search_catalog` | Reads the parts catalog from Google Sheets |
| `check_order` | Finds an order by its number |
| `escalate_to_manager` | Sends the request to the managers' Telegram group |
| `create_return` | Adds a return request to the sheet |

Every answer is logged to the "Журнал" (Log) sheet. If the AI fails, the customer gets a polite "short technical pause" message instead of silence.

**Manager branch.** When a manager replies to the bot's request in the group, a Code node pulls the customer's chat ID out of the original message, the bot sends the answer to the customer and confirms in the group.

### 3. `daily-report.json`: Daily report

Every day at 7 PM it reads today's rows from the log, counts chats and unique customers, asks Gemini to write a short summary and posts it to the managers' group.

## Tech stack

- **n8n** (self-hosted)
- **Telegram Bot API**
- **Google Gemini** (chat model and embeddings, free key from Google AI Studio)
- **OpenRouter** (free fallback model)
- **Google Sheets** (catalog, orders, returns, log)
- **Google Docs** (knowledge base)
- **n8n Simple Vector Store and Simple Memory** (in-memory)

## How to run it yourself

### What you need

- A running n8n instance with a public HTTPS address, so Telegram can reach the webhook (for a local setup, a tunnel like ngrok works).
- A Telegram bot from @BotFather and a Telegram group for managers with the bot added to it.
- A Google AI Studio API key (Gemini) and an OpenRouter API key.
- Google OAuth access for Sheets and Docs.

### 1. Prepare the data

Create a Google Sheet with these tabs (the names are in Ukrainian because the workflows refer to them):

| Tab | Columns |
|---|---|
| Каталог (Catalog) | SKU, Назва, Категорія, Бренд, Марка авто, Модель, Роки, Ціна грн, В наявності шт, Доставка якщо немає |
| Замовлення (Orders) | Номер замовлення, Дата, Клієнт, Телефон, Товари (SKU × к-сть), Сума, Статус, Доставка, ТТН, Місто, Коментар |
| Повернення (Returns) | Дата заявки, Номер замовлення, Клієнт, Telegram chat_id, Товар (SKU), Причина, Стан товару, Бажане рішення, Статус |
| Журнал (Log) | Дата, chat_id, Клієнт, Питання клієнта, Відповідь бота |

Then create a Google Doc with your store rules: about the store, how to pick a part, payment, delivery, returns, warranty and FAQ.

### 2. Create credentials in n8n

Telegram, Google Gemini (PaLM) API, OpenRouter, Google Sheets OAuth2 and Google Docs OAuth2.

### 3. Import the workflows

In n8n go to **Workflows → Import from File** and import all three JSON files.

### 4. Replace the placeholders

The files contain placeholders instead of real values. Replace them with your own:

| Placeholder | What to put there | Where |
|---|---|---|
| `YOUR_GOOGLE_SHEET_ID` | Your Google Sheet. Easiest way: pick the sheet and tab from the list in each Google Sheets node | `check_order`, `search_catalog`, `create_return`, "Запис у журнал", "Get row(s) in sheet" |
| `YOUR_GOOGLE_DOC_ID` | Link to your knowledge base Google Doc | "Get a document" |
| `YOUR_MANAGERS_GROUP_CHAT_ID` | Chat ID of your managers' group (a negative number). You can see it in the Telegram Trigger output after sending a message to the group | `escalate_to_manager`, "If", "Підтвердження менеджеру", "Звіт менеджерам" |
| `YOUR_TELEGRAM_CREDENTIAL_ID` | Select your Telegram credential | All Telegram nodes |
| `YOUR_GEMINI_CREDENTIAL_ID` | Select your Gemini credential | Gemini chat models and embeddings |
| `YOUR_OPENROUTER_CREDENTIAL_ID` | Select your OpenRouter credential | "OpenRouter Chat Model" |
| `YOUR_GOOGLE_SHEETS_CREDENTIAL_ID` | Select your Google Sheets credential | All Google Sheets nodes |
| `YOUR_GOOGLE_DOCS_CREDENTIAL_ID` | Select your Google Docs credential | "Get a document" |

You don't need to type credential IDs by hand: just open each node and pick your credential from the dropdown.

### 5. Load the knowledge base and start the bot

1. Open the knowledge base loader and click **Execute workflow**.
2. Activate the AI consultant and the daily report workflows.
3. Send `/start` to your bot.

> **Important:** the vector store lives in n8n's memory. **Every time n8n restarts, run the knowledge base loader again**, otherwise the bot won't find the knowledge base.

## Demo limitations

- The vector store and chat memory are kept in RAM, so they're wiped when n8n restarts.
- If Google OAuth is in "Testing" mode, access expires every 7 days and has to be reconnected.
- The AI agent doesn't see the managers' replies, because they bypass its memory.
- Free models can be slow in the evening and sometimes return "503 high demand".

For a real store you'd want a VPS with its own domain running 24/7, persistent memory (Postgres or Redis), a persistent vector store (Supabase or Pinecone), manager replies saved to the agent's memory, integration with a real CRM or e-commerce platform, and a paid model for stability.

## Note

All store data (catalog, orders, names, phone numbers, tracking numbers and store policies) is fictional and made up for this demo.
