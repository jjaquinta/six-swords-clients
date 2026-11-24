# Six Swords client API, version 1

Six Swords is a text adventure: you send what the player typed or said, and get back what the game
replies. That is the whole API. Everything the player can do in the game, they do by sending text.

Base URL:

```
https://maincastle.serveminecraft.net/tsatsatzu/sixswords
```

| Call | What it does |
|---|---|
| `GET  {base}/play` | The ready-made web client (the page in [`examples/web`](../examples/web)) |
| `POST {base}/play/say` | Play one turn |
| `GET  {base}/play/commands` | Every command the game understands |

## POST /play/say

Request body: `application/x-www-form-urlencoded`, UTF-8.

| Field | Required | Meaning |
|---|---|---|
| `text` | no | What the player typed or said, up to 300 characters (longer is cut). Blank or missing: see below. |
| `player` | no | The player ID from an earlier reply. Apps can send the header instead (below). |
| `format` | no | `ssml` to get the reply as speech markup as well as text. |
| `include` | no | `state` to get structured data about the player's situation as well ([preview](#state-preview)). |

Request headers (all optional):

| Header | Meaning |
|---|---|
| `X-SixSwords-Player` | The player ID from an earlier reply. Takes priority over the `player` field and the cookie. |
| `Accept` | Naming `application/ssml+xml` has the same effect as `format=ssml`. |

Response: `application/json`, UTF-8.

```json
{
  "reply": "Surrounding you is Light Forest. There is a track going from North to South. ...",
  "ended": false,
  "player": "KfRF0fkJGElFmkBI6qlRZg",
  "ssml": "<speak>Surrounding you is Light Forest. ...</speak>"
}
```

| Field | Meaning |
|---|---|
| `reply` | The game's reply as plain text. Show it, or have it read out. May contain line breaks. |
| `ended` | `true` if the player ended the game (for example by saying "stop"). The next turn starts a new game for the same player. |
| `player` | The player's ID. Keep it and send it with every later turn. Missing only when a new player was refused (429). |
| `ssml` | Only when asked for. The reply as the game speaks it on Alexa: [SSML](https://www.w3.org/TR/speech-synthesis11/) wrapped in `<speak>`, with `<audio>` clips (music, the "more" tone) and `<break>`s. |
| `state` | Only when asked for (`include=state`). See [State (preview)](#state-preview). |

### Turns

- **First turn.** Send a blank `text` with no player ID. The game makes a new player, launches, and
  returns the welcome text and the new `player` ID.
- **Later turns.** Send what the player typed, with the ID.
- **Blank text, known player.** Repeats the last reply. Useful when an app comes back from the
  background or a page reloads. The first turn after a long pause (30 minutes) launches the game
  again instead, so expect a welcome back.
- **"more".** Some replies end with an offer of more detail (on Alexa, a short tone). Send `more` to
  get it.
- **Help.** Send `help` for help on the last thing the player did; send it twice to go to the help
  office in the game.

### Identity

The player ID is a random 22-character string (`A-Z a-z 0-9 _ -`). Whoever holds it is that player,
so store it like a password: in app storage, not in logs or URLs. There are no user names or
passwords; a player who wants the same character on another device, or on Alexa, uses the
in-game account linking (say `help` twice, then go to Support).

The web page uses a cookie instead (`sixswords_player`, HttpOnly). An app doesn't need it: send the
ID in the header.

**Browser clients on another site** can't send the `X-SixSwords-Player` header (the server only
allows the `Content-Type` and `Authorization` headers across sites) and won't get the cookie. Send
the ID in the `player` field instead.

### Status codes

| Code | Meaning | `reply` |
|---|---|---|
| 200 | A turn was played | The game's reply |
| 429 | Too many new players from your address in the last hour | A message to show the player |
| 503 | The server is holding as many games as it will | A message to show the player; try again in a few minutes |
| 500 | The turn failed in the game | A message to show the player; the next turn starts afresh |
| 405 | Wrong method (only POST) | None |

Every error that has a body is JSON in the same shape, so a client can show `reply` whatever the
status.

### Limits

- 20 new players per address per hour. An ID the server has never seen counts as a new player.
- Games are dropped from memory after 30 minutes without a turn. Nothing is lost: the player's
  progress is kept in the game's database.
- Turns for all players, including Alexa's, are taken one at a time. A turn usually takes well
  under a second; send the next turn when the last reply has arrived.

## State (preview)

With `include=state`, each reply also carries a `state` object: the player's situation after the
turn, so a client can build menus and displays without reading them out of the text.

**Preview:** names and shapes may still change while the first clients try it. Changes will be
announced in this repository. When it settles it becomes part of version 1, with the same promise
as the rest. Fields are left out when they don't apply (no `question` when nothing is asked).

```json
"state": {
  "mode": "choice",
  "more": false,
  "commands": ["about", "east", "help", "look", "north", "south", "west", "who"],
  "exits": [],
  "suggestions": ["north", "south", "east", "west"],
  "location": {
    "coords": {"x": 1015, "y": 946, "z": 2},
    "terrain": "Light Forest",
    "place": "Help Office",
    "room": "Tool Shop"
  },
  "question": {
    "kind": "choice",
    "text": "What item do you want to buy?",
    "choices": [{"index": 0, "name": "Pocket Mansion", "current": true}, {"index": 1, "name": "Shovel"}],
    "current": 0,
    "answers": [{"say": "north", "does": "select"}, {"say": "south", "does": "cancel"},
                {"say": "east", "does": "next"}, {"say": "west", "does": "previous"}]
  },
  "menu": {
    "title": "Buy:",
    "items": [{"title": "Shovel", "description": "Shovel for 200", "say": "buy Shovel"}]
  },
  "party": [{"name": "Paul", "hp": 2, "maxHp": 2, "level": 1, "dead": false, "active": true}],
  "gold": 0,
  "opponents": [{"name": "Bat", "hp": 3, "maxHp": 4, "dead": false}]
}
```

| Field | Meaning |
|---|---|
| `mode` | `explore`, `combat`, `question` (yes or no), `direction` (pick a direction) or `choice` (pick from a list). |
| `more` | `true` if the reply offers more detail: send `more` to get it. |
| `commands` | The commands that make sense right now, as IDs from [`/play/commands`](#get-playcommands). |
| `exits` | The directions the party can move in. Empty in a fight. |
| `suggestions` | A few commands the game would suggest now. |
| `location.coords` | Where the party is on the map. `z` is the world (dimension). |
| `location.terrain` | The terrain, as the game names it. |
| `location.place`, `location.room` | When inside a place (a town, a shop, the help office), its name and the room's. |
| `question` | A question waiting for an answer. `kind` is `yesno`, `direction` or `choice`. `answers` lists what to send (`say`) and what it does (`does`). For a `choice`, the game steps through `choices` one at a time: `current` is the one selected now, `east`/`west` move, `north` takes it, `south` cancels. |
| `menu` | Things on offer here (a shop's goods, people to hire). Each item's `say` is the command for it. |
| `party` | The companions: hit points now and at full, level, whether dead, and which one is active. |
| `gold` | The party's gold. |
| `opponents` | Present when the party has met monsters, in a fight or before one: name and hit points of each. |

## GET /play/commands

Every command the game understands, generated from the game's own command model, so it changes when
the game does. JSON:

```json
{
  "language": "en_US",
  "commands": [
    {
      "id": "buy",
      "help": "To buy something, say \"buy\".",
      "phrasings": ["buy", "pick up", "get", "purchase", "buy {what}", "purchase {what}"],
      "slots": [{"name": "what", "accepts": "items", "values": ["Arrow", "Banded Armor", "..."]}]
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `id` | The command's ID, as used in `state.commands`. Sending the ID itself works for commands without slots. |
| `help` | A sentence explaining it, when the game has one. |
| `phrasings` | Ways to say it that the game understands. `{name}` marks a slot to fill in. |
| `slots[].accepts` | `companion` (one of the party's names, see `state.party`), `text` (anything), or the name of a list given in `values` (`items`, `spells`, `types`, `direction`). |

Typed text is matched against the phrasings, so it must be close to one of them.

## Examples

```sh
B=https://maincastle.serveminecraft.net/tsatsatzu/sixswords

# first turn: a new player
curl -s -X POST -d "text=" $B/play/say
# {"ended":false,"player":"KfRF0fkJGElFmkBI6qlRZg","reply":"Welcome to Six Swords, ..."}

# later turns, with speech markup
curl -s -X POST -H "X-SixSwords-Player: KfRF0fkJGElFmkBI6qlRZg" \
     -d "text=look" -d "format=ssml" $B/play/say
```

Kotlin with OkHttp:

```kotlin
val body = FormBody.Builder().add("text", typed).build()
val request = Request.Builder()
    .url("$base/play/say")
    .header("X-SixSwords-Player", playerId)   // leave out on the very first turn
    .post(body)
    .build()
client.newCall(request).execute().use { resp ->
    val json = JSONObject(resp.body!!.string())
    playerId = json.optString("player", playerId)
    show(json.getString("reply"))
}
```

## Versioning

This is version 1. Within version 1:

- fields and optional parameters may be **added** (ignore fields you don't know);
- nothing will be renamed, removed or change meaning.

A change that would break that becomes version 2, at a new path, with version 1 kept running while
clients move. The [state](#state-preview) is the exception while it is a preview.

## Accessibility

Many Six Swords players are blind or visually impaired. If you write a client, please make it work
with a screen reader (TalkBack on Android, VoiceOver on iOS): announce each reply once, keep focus
where the player left it, and give every control a name. The example page shows one way.
