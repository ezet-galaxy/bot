# Bot

The `bot` package provides Telegram Bot API integration for [Flame](https://github.com/shoya-129/flame).

It simplifies Telegram bot development by providing functions for sending replies and configuring Telegram webhooks. The package works together with the [flamer](https://github.com/shoya-129/flamer) HTTP server package to receive Telegram updates through Flame HTTP routes.

## Features

- Send replies to Telegram users with `bot.reply`.
- Configure Telegram webhooks with `bot.setWebhook`.
- Handle Telegram updates as Flame `Formula` values.
- Integrate with `flamer` for HTTP webhook endpoints.
- Keep bot tokens and webhook URLs in environment variables.
- Avoid manually constructing Telegram Bot API HTTP requests.

## Installation

Install the `flamer` HTTP server package:

```bash
flame add https://github.com/shoya-129/flamer
```

Then install the Telegram bot package:

```bash
flame add https://github.com/ezet-galaxy/bot
```

Import the packages in your Flame application:

```flame
import bot
import flamer
```

You can also use Flame standard-library modules such as `std.env` and `std.json`:

```flame
import std.env
import std.json
```

## Environment Variables

The Telegram Bot API token should not be hard-coded into the application.

Set the bot token and public webhook URL as environment variables:

```env
BOT_TOKEN=your-telegram-bot-token
WEBHOOK_URL=https://your-domain.com/webhook
```

Read them from the environment:

```flame
import std.env

let bot_token: String? = env.get("BOT_TOKEN")
let webhook_url: String? = env.get("WEBHOOK_URL")
```

`WEBHOOK_URL` should point to the public HTTPS endpoint where Telegram will send updates.

For example:

```text
https://example.com/webhook
```

## API

### `bot.reply`

```flame
bot.reply(
    update: &Formula,
    token: &String,
    replyText: String
)
```

Sends a text message to the Telegram chat associated with an incoming update.

#### Parameters

| Parameter   | Type       | Description                             |
| ----------- | ---------- | --------------------------------------- |
| `update`    | `&Formula` | Telegram update received by the webhook |
| `token`     | `&String`  | Telegram Bot API token                  |
| `replyText` | `String`   | Text that should be sent to the user    |

 **Returns**
 `
Formula`

The returned `Formula` contains the response from the Telegram Bot API.

### `bot.setWebhook`

```flame
bot.setWebhook(
    token: &String,
    webhookUrl: &String
)
```

Registers a webhook URL with Telegram.

#### Parameters

| Parameter    | Type      | Description                                |
| ------------ | --------- | ------------------------------------------ |
| `token`      | `&String` | Telegram Bot API token                     |
| `webhookUrl` | `&String` | Public HTTPS URL that Telegram should call |

**Returns** `Formula`

The returned `Formula` contains Telegram's webhook configuration response.

## Webhook Architecture

The bot package separates **Telegram webhook configuration** from **Flamer server configuration**.

The webhook registration happens **outside** the `@Flamer` `main()` function.

The `@Flamer` function is responsible for configuring and starting the HTTP server, while `bot.setWebhook()` is responsible for telling Telegram where to send updates.

```text
                 Telegram
                    │
                    │ Telegram Update
                    ▼
            /webhook endpoint
                    │
                    ▼
              Flamer Server
                    │
                    ▼
            Flame webhook()
                    │
                    │ bot.reply()
                    ▼
             Telegram Bot API
```

### Responsibility separation

**Outside** `@Flamer`

```flame
let webhookRes = await bot.setWebhook(
    &bot_token,
    &webhook_url
)
```

This configures Telegram's webhook.

**Inside** `@Flamer`

```flame
@Flamer(port: 3000)
async fn main() {
    flamer.post("/webhook", webhook)

    await flamer.listen()
}
```

This configures the Flame HTTP server and registers the webhook route.

### Important

Do not put `bot.setWebhook()` inside the `@Flamer` `main()` function.

The intended application structure is:

```text
Application startup
       │
       ├── bot.setWebhook()
       │       │
       │       └── Configure Telegram
       │
       └── @Flamer main()
               │
               ├── flamer.post()
               │
               └── flamer.listen()
```

## Handling Telegram Updates

Telegram sends webhook updates as JSON.

The `flamer` route receives the request body as a `Formula`. The application can parse that body using `std.json`.

```flame
async fn webhook(body: Formula) -> Formula {
    let data: Formula? = json.parse(body)

    let text: String? = data.message.text
    let chat_first_name: String? = data.message.chat.first_name

    println($"Received from {chat_first_name}: {text}")

    return {
        ok: true
    }
}
```

The parsed Telegram update can then be passed to `bot.reply`.

## Command Handling

Flame's pattern matching can be used to implement Telegram commands:

```flame
match text {
    "/start" => {
        await bot.reply(
            &data,
            &bot_token,
            $"Hello {chat_first_name}"
        )
    }

    "/help" => {
        await bot.reply(
            &data,
            &bot_token,
            "Available commands: /start, /help"
        )
    }

    _ => {
        await bot.reply(
            &data,
            &bot_token,
            "Hola, I have no answer for this!"
        )
    }
}
```

## Complete Example

The following example demonstrates the intended architecture.

```flame
import std.net.http
import std.json
import std.env
import flamer
import bot

let bot_token: String? = env.get("BOT_TOKEN")
let webhook_url: String? = env.get("WEBHOOK_URL")

async fn webhook(body: Formula) -> Formula {
    let data: Formula? = json.parse(body)

    let text: String? = data.message.text
    let chat_first_name: String? = data.message.chat.first_name

    println($"Received from {chat_first_name}: {text}")

    match text {
        "/start" => {
            await bot.reply(
                &data,
                &bot_token,
                $"Hello {chat_first_name}"
            )
        }

        _ => {
            await bot.reply(
                &data,
                &bot_token,
                "Hola, I have no answer for this!"
            )
        }
    }

    return {
        ok: true
    }
}

// Configure the Telegram webhook outside @Flamer.
let webhookRes = await bot.setWebhook(
    &bot_token,
    &webhook_url
)

println($"Webhook setup: {webhookRes?.description}")

// Configure and start the HTTP server.
@Flamer(port: 3000)
async fn main() {
    println("--- Flame Telegram Bot Webhook Server ---")

    // Register the Telegram webhook endpoint.
    flamer.post("/webhook", webhook)

    println("Server listening on port 3000.")
    println($"Point your Telegram webhook to: {webhook_url}")

    await flamer.listen()
}

await main()
```

## Startup Flow

When the application starts, the flow is:

### 1. Read configuration

```flame
let bot_token: String? = env.get("BOT_TOKEN")
let webhook_url: String? = env.get("WEBHOOK_URL")
```

### 2. Configure Telegram

```flame
let webhookRes = await bot.setWebhook(
    &bot_token,
    &webhook_url
)
```

Telegram is now configured to send updates to the specified webhook URL.

### 3. Configure the Flamer route &  Start the server

```flame
@Flamer(port: 3000)
async fn main() {
    flamer.post("/webhook", webhook)

    await flamer.listen()
}
```

- The `/webhook` endpoint is registered with the Flamer server.
- Flamer starts listening for incoming HTTP requests.

### 5. Receive Telegram updates

Telegram sends updates to:

```text
https://your-domain.com/webhook
```

Flamer invokes the `webhook` handler.

### 6. Reply to the user

The handler can respond using:

```flame
await bot.reply(
    &data,
    &bot_token,
    "Hello!"
)
```

## Package Requirements

A Telegram bot application using this package requires:

- `bot`
- `flamer`
- `std.json`
- `std.env`
- A Telegram bot created through BotFather
- A publicly reachable HTTPS webhook endpoint
- For local development, use ngrok or a Cloudflare Tunnel to expose your localhost and use the generated HTTPS URL as the Telegram webhook URL

Install the external packages with:

```bash
flame add https://github.com/shoya-129/flamer
flame add https://github.com/ezet-galaxy/bot
```

## Repositories

- Flamer: [https://github.com/shoya-129/flamer](https://github.com/shoya-129/flamer)
- Bot: [https://github.com/ezet-galaxy/bot](https://github.com/ezet-galaxy/bot)

The key architectural rule is:

> `bot.setWebhook()` **configures Telegram outside** `@Flamer`**, while** `@Flamer` **configures and runs the HTTP server.**
