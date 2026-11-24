# Six Swords clients

[Six Swords](https://maincastle.serveminecraft.net/tsatsatzu/sixswords/play) is a fantasy adventure
played entirely in words: you lead a party of up to six mercenaries, explore, fight monsters and
bring back their gold. It started as an Alexa skill, and many of its players are blind or visually
impaired.

This repository is for people who want to write their own client for it: an Android or iOS app, a
desktop program, a MUD-style terminal client. The game itself runs on our server; a client sends
what the player types and shows (or speaks) what comes back.

- **[API reference](docs/API.md)**: the two calls, identity, speech markup (SSML), errors, limits.
- **[Example web client](examples/web/play.html)**: the page served at `/play`. Plain HTML and
  JavaScript, built for screen readers.
- **[Fair use](FAIR-USE.md)**: please read before you publish a client.

## Quick start

```sh
curl -s -X POST -d "text=" https://maincastle.serveminecraft.net/tsatsatzu/sixswords/play/say
```

The reply has the welcome text and a `player` ID; send that ID back with every later turn
(see the [API reference](docs/API.md)).

## Questions and requests

Open an issue here for questions about the API, problems with it, or things a client needs that it
doesn't give yet. Bugs in the game itself are welcome too.

## Licences

- Code (the example client, snippets): [MIT](LICENSE).
- Documentation: [CC BY 4.0](LICENSE-docs.md).
- The game, its text and its sounds are not in this repository and are not covered by these
  licences: see [Fair use](FAIR-USE.md).
