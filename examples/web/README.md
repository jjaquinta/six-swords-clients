# Example web client

`play.html` is the page the server itself serves at
`https://maincastle.serveminecraft.net/tsatsatzu/sixswords/play`. It is a copy: the server's copy is
the original, and this one is updated from it.

What it shows:

- one plain HTML file, no framework or build step;
- a labelled command box that keeps focus, and named buttons for frequent commands;
- each reply written into a hidden `role="status"` live region, so screen readers read it once,
  and kept in a "Story so far" list for reading back;
- turns sent one at a time, queued if the player is quicker than the server;
- light and dark colours from the device setting, and a layout that fits a phone.

It calls `play/say` relative to its own address and relies on the cookie for identity, because it is
served from the game's own site. To run a copy from anywhere else, change the URL in `fetch(...)` to
the full `https://maincastle.serveminecraft.net/tsatsatzu/sixswords/play/say`, keep the `player`
value from each reply (for example in `localStorage`), and send it back in a `player` form field:
see [Identity](../../docs/API.md#identity).
