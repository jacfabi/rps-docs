# Client Guide

This document only covers what a bot author needs in order to build and run a client for this game.

Your client can be written in any language. The only requirement is that it fulfills the bot contract described below.

## Game Rules

Valid choices:

- `rock`
- `fire`
- `scissors`
- `sponge`
- `paper`
- `air`
- `water`

Winning relationships:

- Rock beats Fire, Scissors, Sponge
- Fire beats Scissors, Paper, Sponge
- Scissors beats Air, Paper, Sponge
- Sponge beats Paper, Air, Water
- Paper beats Air, Rock, Water
- Air beats Fire, Rock, Water
- Water beats Rock, Fire, Scissors

## Bot Contract

Your bot must expose this HTTP endpoint:

`POST /play`

The server will send a JSON body with the full history of the current battle from your bot's perspective:

```json
{
  "history": [
    {
      "my_choice": "rock",
      "opponent_choice": "scissors",
      "outcome": "win"
    },
    {
      "my_choice": "fire",
      "opponent_choice": "water",
      "outcome": "loss"
    }
  ]
}
```

`history` is an ordered list of all games played so far in the current battle. It is empty on the first game.

Each entry contains:

| Field | Type | Description |
|---|---|---|
| `my_choice` | string or null | The choice your bot made (null if your bot forfeited) |
| `opponent_choice` | string or null | The choice the opponent made (null if they forfeited) |
| `outcome` | string | Result from your perspective (see below) |

Possible `outcome` values:

| Value | Meaning |
|---|---|
| `win` | You won this game |
| `loss` | You lost this game |
| `draw` | Both choices tied |
| `forfeit_win` | Opponent forfeited, you win |
| `forfeit_loss` | You forfeited (your bot failed to respond), opponent wins |
| `forfeit_both` | Both bots forfeited |

Your bot must return JSON in this format:

```json
{
  "choice": "rock"
}
```

Where `choice` is one of:

- `rock`
- `fire`
- `scissors`
- `sponge`
- `paper`
- `air`
- `water`

Requirements:

- your bot must respond to `POST /play`
- your response must be valid JSON
- your JSON must contain a `choice` field
- `choice` must be one of the 7 valid values above
- you may ignore the `history` body entirely if you don't need it

The game host will call:

```text
POST {your-public-endpoint}/play
```

So if your public bot URL is:

```text
https://xyz789.ngrok-free.app
```

The host will request:

```text
POST https://xyz789.ngrok-free.app/play
```

## Example Client In Python

Python is only an example here. You may use any language or framework.

```python
from random import choice

from fastapi import FastAPI
from pydantic import BaseModel
import uvicorn

app = FastAPI()

CHOICES = [
    "rock",
    "fire",
    "scissors",
    "sponge",
    "paper",
    "air",
    "water",
]


class HistoryEntry(BaseModel):
    my_choice: str | None
    opponent_choice: str | None
    outcome: str


class PlayRequest(BaseModel):
    history: list[HistoryEntry] = []


@app.post("/play")
def play(request: PlayRequest):
    # request.history contains all previous games — use it or ignore it
    return {"choice": choice(CHOICES)}


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=9001)
```

## Run Your Client

How you run your client depends on the language and framework you use.

For the Python example above:

Create a virtual environment and install dependencies:

```bash
python3 -m venv .venv
.venv/bin/pip install fastapi uvicorn
```

Run your bot:

```bash
.venv/bin/python external_bot.py
```

That starts the bot locally on:

```text
http://localhost:9001
```

## Registering Your Bot

You need the host's public server URL, for example:

```text
https://abc123.ngrok-free.app
```

Then register with a name, your bot endpoint, and a cheer:

```bash
curl -X POST "https://abc123.ngrok-free.app/register" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MyBot",
    "endpoint": "https://your-public-bot-url.example",
    "cheer": "YOU ARE GETTING SMOKED"
  }'
```

Notes:

- `name` is your bot name shown in the UI
- `endpoint` must be the public base URL of your bot
- `cheer` is the taunt shown when your battle starts
- the game service will call `POST {endpoint}/play`

## Using ngrok

If your bot is running locally on your laptop, `localhost` is not reachable by the game host. You need to expose it publicly.

If your bot runs on port `9001`, start ngrok like this:

```bash
ngrok http 9001
```

ngrok will show a public URL like:

```text
https://xyz789.ngrok-free.app
```

Use that URL as your `endpoint` when registering.

Example:

```bash
curl -X POST "https://abc123.ngrok-free.app/register" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "MyBot",
    "endpoint": "https://xyz789.ngrok-free.app",
    "cheer": "YOU CANNOT HANDLE THIS BOT"
  }'
```

Important:

- do not register `http://localhost:9001`
- keep your bot process running
- keep your ngrok tunnel running
- if ngrok restarts and gives you a new URL, register again with the new URL

## Helper Registration Script

This helper is also just an example. You may register your bot using any HTTP client or language.

```python
import json
import sys
from urllib.request import Request, urlopen

if len(sys.argv) != 5:
    print("Usage: python register_bot.py <server_url> <bot_name> <bot_public_url> <cheer>")
    raise SystemExit(1)

server_url = sys.argv[1].rstrip("/")
bot_name = sys.argv[2]
bot_public_url = sys.argv[3].rstrip("/")
cheer = sys.argv[4]

payload = json.dumps({
    "name": bot_name,
    "endpoint": bot_public_url,
    "cheer": cheer,
}).encode("utf-8")

request = Request(
    f"{server_url}/register",
    data=payload,
    headers={"Content-Type": "application/json"},
    method="POST",
)

with urlopen(request) as response:
    print(response.read().decode("utf-8"))
```

Usage:

```bash
python3 register_bot.py \
  https://abc123.ngrok-free.app \
  MyBot \
  https://xyz789.ngrok-free.app \
  "YOU JUST ENTERED MY ARENA"
```

## Quick Checklist

1. Implement the bot contract: `POST /play`
2. Return valid JSON with a valid `choice`
3. Run your bot locally using your language/framework of choice
4. Expose it with `ngrok http <port>`
5. Register it with the host using the public ngrok URL
6. Keep both the bot and ngrok running during the event
