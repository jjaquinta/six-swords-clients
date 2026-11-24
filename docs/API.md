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

## POST /play/say

Request body: `application/x-www-form-urlencoded`, UTF-8.

| Field | Required | Meaning |
|---|---|---|
| `text` | no | What the player typed or said, up to 300 characters (longer is cut). Blank or missing: see below. |
| `player` | no | The player ID from an earlier reply. Apps can send the header instead (below). |
| `format` | no | `ssml` to get the reply as speech markup as well as text. |

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
clients move. Planned additions include structured data alongside the text (where the player is,
the exits, the party's health, a question waiting for an answer), so rich clients don't have to read
it out of the text.

## Accessibility

Many Six Swords players are blind or visually impaired. If you write a client, please make it work
with a screen reader (TalkBack on Android, VoiceOver on iOS): announce each reply once, keep focus
where the player left it, and give every control a name. The example page shows one way.
