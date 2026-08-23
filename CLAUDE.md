# Working on HTML City

A first-person walk through any real city, built live from OpenStreetMap. See
`README.md` for what it does and why it is built the way it is. This file is
about working on it.

## Hard constraints

- **One file.** Everything is `index.html` — markup, CSS, and one ES module. No
  build step, no bundler, no `node_modules`. Do not introduce any.
- **No API keys, no accounts.** Every data source is free and CORS-open. If a
  feature needs a key, it does not go in.
- **three.js is vendored** in `vendor/` and wired up with an import map. Do not
  put a CDN back in the runtime path.
- **Degrade, never break.** Each external service is optional: no DEM means flat
  ground, no speech voices means subtitles only, no font for a script means a
  romanisation. Nothing throws because a service is down.
- `running.arms.gif` is reference art and is gitignored. Leave it that way.

## Layout of index.html

Roughly in order. Line numbers drift; the banner comments are the landmarks.

| section | what lives there |
|---|---|
| config, projection | constants, local-metre projection about the landing point |
| three.js, the real sun | renderer, sky, solar position, sky/light keyframes |
| civil time | IANA zone lookup, the clock |
| textures, geometry accumulator | canvas textures, `Buf`, `addFlat`/`addBuilding`/`addRibbon`/`addSkirt` |
| OSM | Overpass query, tag parsing, ring stitching |
| world state | `colliders`, `roads`, `bridges`, the spatial grid |
| terrain | Terrarium DEM fetch, `groundAt` |
| speech | language map, phrase tables, romanisation, `say` |
| tile cache, tile grid | IndexedDB, globe-pinned tiles, `buildTile` |
| player, view arms | movement, collision, arms, weapons |
| people | crowd, health, faces, chase/flee, speech triggers |
| main loop, UI | `frame()`, start screen, geocoding |

## Verifying a change

`index.html` is not importable, so extract the module and parse it:

Set up once (esprima is a pure-Python JS parser; there is no node here):

```bash
python3 -m venv /tmp/hc-venv && /tmp/hc-venv/bin/pip -q install esprima
```

Then, after every edit:

```bash
python3 -c "
import re;print(re.search(r'<script type=\"module\">(.*?)</script>',
open('index.html',encoding='utf-8').read(), re.S).group(1))" > /tmp/hc-module.js

/tmp/hc-venv/bin/python - <<'EOF'
import esprima, collections
src = open('/tmp/hc-module.js', encoding='utf-8').read()
tree = esprima.parseModule(src, {'loc': True})        # syntax
names = collections.defaultdict(list)                 # and redeclaration
for nd in tree.body:
    if nd.type == 'VariableDeclaration':
        for d in nd.declarations:
            if d.id.type == 'Identifier':
                names[d.id.name].append(d.loc.start.line)
    elif nd.type in ('FunctionDeclaration', 'ClassDeclaration') and nd.id:
        names[nd.id.name].append(nd.loc.start.line)
dupes = {k: v for k, v in names.items() if len(v) > 1}
print('SYNTAX OK;', dupes or f'no duplicates among {len(names)} bindings')
EOF
```

**The parse alone is not enough**, which is why the snippet does both. Duplicate
`const` at module scope is a semantic early error, not a syntax error, and it
kills the whole module silently — the symptom is `city is not defined` and a
blank page. It has happened here, `ANGRY` the colour against `ANGRY` the phrase
table, and the parser sailed straight past it.

Note esprima predates optional chaining, so `?.` will read as a syntax error.
The code avoids it for that reason.

**Neither check catches a temporal-dead-zone error**, and that one takes the
whole page down: reference a `const` declared further down the module and it
throws at load, `window.city` never appears, and every symptom looks unrelated —
a dead search box, no crowd, nothing rendering. It has happened here, building
the weapon bar from `WEAPONS` before `WEAPONS` was declared. So finish with a
runtime smoke test that proves the module actually executed:

```js
typeof window.city === 'object'          // the module reached its last line
document.querySelectorAll('#wbar span')  // things built from later constants
!!city.renderer.getContext()             // WebGL came up
```

Load the page headless, wait a few seconds, assert those and that no exception
was thrown. It takes about five seconds and catches a class of fault that no
amount of static checking will.

For anything visual or behavioural, drive a real browser over the DevTools
protocol: launch headless Chrome with `--remote-debugging-port`, attach by
WebSocket, and read state out of `window.city` (see below). Screenshots are
useful but measurements are better — count what actually happened rather than
squinting at a picture.

## Traps that have already cost time

- **`dt` is clamped to 0.1 s** so a stall cannot teleport you. Every cooldown
  and timer therefore stretches proportionally below 10 fps. Headless software
  rendering runs at about 1 fps here, so a 0.34 s reload takes several real
  seconds and a 3.5 s timer takes half a minute. Several "bugs" were this.
- **Headless readings can be a stale frame.** State set from the console may not
  be reflected in `count`/visibility until frames actually tick. Re-read after
  waiting, and turn shadows off to speed the renderer up.
- **Overpass firewalls IPs that lean on it.** A refused TCP connection surfaces
  in the browser as a bare `Failed to fetch`, which looks like being offline but
  is not — `curl` will still get 200. Cache and pace; do not hammer it in tests.
- **`| tail` withholds everything until EOF**, so a test killed by `timeout`
  prints nothing at all. Redirect to a file and read the file.
- **Stale Chrome processes accumulate** across killed tests and wreck frame
  rates. `pgrep -x chrome` to check, and kill by PID — a `pkill -f` pattern can
  match your own shell.
- **`ShapeUtils.triangulateShape` needs real `Vector2`s** (it calls `.equals`)
  and mutates its input by popping duplicate end points, so build any flat index
  list *after* it runs, not before.
- **Country codes are not language codes.** `ar` is Argentina, `sv` is El
  Salvador, `se` is Sweden, `no` is Norway. Getting this wrong is silent.
- **Where you insert code matters as much as what it says.** The module is one
  long scope, so anything built at load time must sit below the constants it
  reads. Prefer to put new setup next to the data it depends on.
- **The render loop calls `pump()` every frame**, so a delay expressed as a
  `setTimeout` gets raced by the next frame. Gate on a timestamp instead.

## Untrusted input

Everything the page displays from the network is third-party data, and two
sources are **editable by the public**: OpenStreetMap tags and the Nominatim
`display_name` built from them. A place can be named anything, including markup.

So text from OSM, Nominatim, timeapi or the DEM must reach the DOM as
`textContent` or as constructed nodes, never as `innerHTML`. This has already
bitten once: search suggestions built their two-line layout with a template
literal, and a place called `<img src=x onerror=...>` executed in the visitor's
page. Verified exploitable before the fix and inert after.

`innerHTML` is fine for markup you wrote with values you generated — the stats
counter interpolates two numbers — but never for a string that came off the wire.

## Conventions

- Everything is relative to the landing point: `groundAt` returns 0 there, and
  returns 0 everywhere if the DEM failed.
- Land is draped over the terrain by adaptive tessellation; **water is not** —
  it is level per body, because water does not run up a hill.
- Positions are `{x, y}` with `y` northing in the 2D world, and three.js gets
  `(x, height, -y)`. Mixing these up is the most common geometry mistake here.
- A person's position comes from `npcPoint(n)` — never re-derive it from the
  path, or the free-moving chasers and the shove offsets get missed.

## The console handle

`window.city` exposes the innards. Useful entry points:

```js
city.player                 // { x, z, y, yaw, pitch }
city.go(52.37, 4.89)        // teleport
city.groundAt(x, z)         // terrain height
city.peopleDebug()          // crowd, moods, boats, despawn reasons, speech counters
city.dem()                  // DEM state
city.speech()               // resolved language and voice
city.missing()              // scripts with no font on this machine
```

`peopleDebug().gone` counts despawns by reason, and `.spx` counts speech
attempts and blocks — reach for those before theorising about why somebody
vanished or said nothing.

## Committing

Commit messages here explain *why* and cite the measurement that settled it —
that is the house style and it has repeatedly paid for itself. Push to `main`.
