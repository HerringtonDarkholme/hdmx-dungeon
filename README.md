# hdmx-dungeon

*The Ember Below*: a turn-based dungeon crawler that runs entirely on the
server, drawn as a terminal in the browser.

Play it at <https://hdmx-dungeon.vercel.app>.

It is written in [hd](https://github.com/HerringtonDarkholme/hd-lang) on the
[hd server SDK](https://github.com/HerringtonDarkholme/hd-server-sdk). The
browser holds no game logic: [htmx](https://htmx.org) posts each key press,
and the server answers with the HTML that the turn changed.

Find the Ember on floor 5 and take it from its Keeper. Monsters hunt you
once they can see you. Kills earn experience, and each level adds health
and makes your blows stronger. Gold buys healing and gear from the merchant
on each floor.

## Run

You need the `hd` command and Node.js 26:

```sh
cargo install hd-lang
```

Then, in this folder:

```sh
hd run
```

Open <http://127.0.0.1:8080/>. Set `HD_SERVER_PORT` to use another port.

## Play

| Key | Action |
| --- | --- |
| Arrow keys, `W` `A` `S` `D` or `H` `J` `K` `L` | Move. Walk into a monster to attack it. |
| `.` or `Z` | Wait one turn. |
| `1` `2` `3` | Buy from the merchant. `Esc` leaves her shop. |
| `Enter` | Start again after the game ends. |

The pad of arrows on the page does the same.

| Symbol | Meaning |
| --- | --- |
| `@` | You |
| `!` | Potion: restores 12 health |
| `$` | Gold |
| `/` | Blade: every blow you land does 1 more |
| `]` | Shield: every blow you take does 1 less |
| `&` | Merchant: walk into her to trade |
| `>` | Stairs down |
| `*` | The Ember |

| Monster | Behaviour |
| --- | --- |
| `r` rat | Weak. |
| `b` bat | Weak, and flies about at random. |
| `x` spider | Quick: moves twice a turn. |
| `s` slime | Tough but slow: acts every other turn. |
| `i` imp | Spits fire from up to five steps away. |
| `G` ghoul | Strong. |
| `O` ogre | Very strong, and slow. |
| `W` wraith | Strong. |
| `K` the Keeper | Guards the Ember on the last floor. |

Deeper floors hold more monsters and stronger kinds.

## How It Works

| File | Job |
| --- | --- |
| [`src/main.hd`](src/main.hd) | Routes, and reading the game a page sends with each request. |
| [`src/game.hd`](src/game.hd) | The rules: moving, fighting, levels, the shop, line of sight. No HTTP, no waiting. |
| [`src/level.hd`](src/level.hd) | Builds a floor of rooms joined by corridors. |
| [`src/save.hd`](src/save.hd) | Packs a game into a short piece of text, and checks one that comes back. |
| [`src/sound.hd`](src/sound.hd) | Builds each sound effect as the bytes of a WAV file. |
| [`src/view.hd`](src/view.hd) | Turns a game into HTML, and holds the page and its styles. |

### A terminal made of text

The screen is 80 columns of one terminal font. Every frame, meter and
dialog is characters, as a terminal program would draw them:

```text
┌─ STATUS ───────────────────┐
│ Floor                  1/5 │
│ Level   ███░░░░░░░       2 │
│ Health  ████████░░   24/30 │
```

- Each piece of the page always fills the same number of cells, so the
  columns stay straight when a number grows. The tests check the width of
  every line.
- The font is cut down to the characters the screen uses, so the box
  characters come from the same font as the letters.
- The text size follows the window, so the 80 columns fit a desktop and
  the 480-pixel square of an embedded page.
- A phone held upright is too narrow for 80 columns. There the frames go,
  the map's 48 columns run from edge to edge, and the status and a
  thumb-sized pad sit under it. A wide touch screen swaps the key list for
  a larger pad.
- CSS adds only the glow, the scanlines and the torch light. The pad of
  arrows is the one part drawn as ordinary buttons.

### Controls

Each control on the page is one htmx element, for example:

```html
<button hx-post="/move/north"
        hx-trigger="click, keydown[key=='ArrowUp'||key=='w'] from:body">↑</button>
```

Requests queue in order (`hx-sync`), so fast key presses are never lost or
shown out of order.

A pad button lights up whenever it sends a request, so pressing a key shows
on the screen as well as clicking does. htmx announces each request on the
element that makes it, and the pad answers with a short animation.

### The server keeps nothing

A visitor's whole game is about 320 bytes of text in a hidden field of the
page. Every request sends it back. The server loads the game from it, plays
one turn, and answers with the new text. Any copy of the program can play
the next turn, which suits platforms that start several workers, such as
Vercel.

Because the game is in the page, each tab plays its own copy, and the game
works where a browser refuses cookies, as it may inside another site. A
cookie holds a second copy, and only a reload reads it: that is how a
reload finds the game again.

The text stays small because it leaves out what the seed can rebuild. A
floor is generated from the game's seed and the floor number, and each
turn's dice from the seed and the turn number.

### A turn sends only what changed

The first load sends the whole page. After that the server draws the page
before and after the turn, and answers with the pieces that differ, each
marked for an htmx out-of-band swap:

```html
<b id="turn" hx-swap-oob="true">        13</b>
<div id="light" class="light" style="--x:21;--y:9" hx-swap-oob="true"></div>
<span id="r9" class="row" hx-swap-oob="true">…</span>
<ol hx-swap-oob="beforeend:#log"><li>You pocket 4 gold.</li></ol>
```

- Each map row and each number in the status panel is its own piece.
- The torch light is one element with two numbers, so walking does not
  redraw every row the light touches.
- New messages are appended to the log. The page keeps the older ones.
- A step into a wall changes nothing, and the answer is an empty `204`.

A turn that changes something answers with about 850 bytes of HTML, of
which 370 are the game's new text. Sending the whole game element takes
about 6,500. The cookie adds another 380 bytes to the response's headers.

### Sound without a script or a file

The server has no sound files. `sound.hd` computes each effect from a few
notes (square, triangle and noise waves) and serves the result as a WAV
file under `/sound/NAME.wav`. A file is 1 to 11 KB, and the browser caches
it.

The page runs no script to play them. A turn that makes a sound includes an
element the browser plays as soon as it arrives:

```html
<div id="sound" hx-swap-oob="true"><audio autoplay src="/sound/coin.wav"></audio></div>
```

The sound setting travels in the page like the game does, so a quiet page
simply gets no such element.

### A buzz on a blow

On a phone, a turn with a blow in it buzzes: a short buzz when you strike,
a longer one when you are hit. The server names the buzz in a response
header, `HX-Trigger: {"rumble":[25,45,70]}`. htmx raises that as an event,
and one line on the page hands it to the browser's vibration call. The
sound switch turns it off as well.

Browsers on Android vibrate. Safari on an iPhone has no vibration call, so
there the game stays still.

### Link previews, and playing inside a post on X

The page carries the tags that sites read to preview a link: a title, a
line about the game, and a picture. The pictures are the two files in
[`assets/`](assets): `preview.jpg`, wide, for most sites, and `card.jpg`,
square, for X. They are screenshots of the game, and the page points at
them on GitHub, because the server has no files of its own.

For X the page asks for a "player card", which plays the game inside the
post, in a 480-pixel square. X decides which sites may show one. If a post
shows no preview at all, set `CARD` at the top of
[`src/view.hd`](src/view.hd) to `summary_large_image`, which shows the wide
picture as an ordinary link.

X shows the game in a frame that may not open dialogs or submit forms.
The game does neither: every control is an htmx request, and "New game"
asks its question in the page.

X no longer has a tool that checks a card. The test is to post the link.

X remembers what it found at an address for about a week. After a change,
share the address with something added, as `/?2`, to make it look again.

## Deploy To Vercel

The game runs as one Vercel function. A task prepares it on your machine,
and Vercel's own tool uploads it:

```sh
hd build
hd run deploy-vercel --engine node
vercel deploy --prebuilt --prod
```

Vercel cannot build this repository itself, so
[`vercel.json`](vercel.json) turns off its deployments from Git pushes. A
push that deployed would replace the game with an empty site.

[`tasks/deploy-vercel.hd`](tasks/deploy-vercel.hd) holds the settings. It
starts four workers in each copy of the function. Any of them can play any
turn, because the game travels with the request.

## Test

```sh
hd test
```

## Limits

- The game's text is not signed. A player can edit it to cheat in their own
  game. The server checks every value, so edited text cannot break it: a
  value no real game could hold starts a new game.
- The message log lives in the page only. Reloading keeps the game and
  clears the log.
- Browsers only play sound on a page the visitor has clicked or typed in,
  which playing the game does. Sound was checked in Chrome only, and a site
  that embeds the game may not let it play sound.
- In a square as small as X's, the text is about 9 pixels high.
- The page loads htmx and its font from public CDNs. Without the font it
  falls back to the system's monospace, whose box characters may not line
  up as well.
