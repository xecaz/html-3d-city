# HTML City

Type an address. Land in it. Walk away down the street.

A single HTML file that pulls a real city out of OpenStreetMap and rebuilds it as
walkable 3D geometry in the browser. Search `Prinsengracht 263, Amsterdam`, press
Enter, and a moment later you are standing on the quay in first person with the
arrow keys under your fingers. Keep walking and the next blocks are fetched and
built in front of you, so there is no edge of the world to bump into.

No build step, no bundler, no `node_modules`, no API keys, no account. One HTML
file of about a thousand lines, plus a vendored copy of three.js — so the only
thing it needs from the network at runtime is the map data itself.

## Run it

```bash
python3 -m http.server 8777
# then open http://127.0.0.1:8777/index.html
```

It is entirely static: there is no server-side process, and your host never
makes an outbound request of its own, because every API call is made by the
visitor's browser. Any static server will do — nginx, Apache, Caddy, S3, GitHub
Pages. It does have to be served over `http://` or `https://` rather than
`file://`, or the module import and the API calls are blocked. Every external
origin it touches is HTTPS, so there is no mixed-content problem on a TLS site.

Jump straight to a spot with a hash:

```
http://127.0.0.1:8777/index.html#52.3752,4.8840
```

The hash follows you as you walk, so any place you find is a shareable link.

## Controls

| | |
|---|---|
| <kbd>↑</kbd> <kbd>↓</kbd> | walk forward / back |
| <kbd>←</kbd> <kbd>→</kbd> | turn |
| <kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd> | walk and strafe |
| mouse | look around — click once to capture the pointer, <kbd>Esc</kbd> to release |
| <kbd>Shift</kbd> | run |
| <kbd>[</kbd> <kbd>]</kbd> | rewind / advance the sun by 15 minutes |
| <kbd>F</kbd> | hold to express an opinion — they notice |

The search box takes a street address, a place name, or a raw `lat, lon` pair.

## Where the data comes from

Everything is public and free, and nothing is scraped.

- **[OpenStreetMap](https://www.openstreetmap.org/copyright)** via the **Overpass
  API** — building footprints, heights, roads, canals, bridges, parks. Requested
  per tile as you move, from three mirrors with automatic failover.
- **[Nominatim](https://nominatim.openstreetmap.org/)** — turns what you type into
  a latitude and longitude.
- **[three.js](https://threejs.org/) r169** — WebGL rendering. Vendored into
  `vendor/` (MIT, licence included) and wired up with an import map, so there is
  no CDN in the runtime path and the page cannot be broken by someone else's
  outage. Swap in the unminified `three.module.js` from the same release if you
  ever want to step through it in a debugger.

A note on Google, since it is the obvious question: the Blender plugin that does
something similar (Blosm) uses OpenStreetMap for its free path. Scraping Google
Maps is against their terms; the legitimate Google route is their paid
Photorealistic 3D Tiles API, which needs a billing-enabled key and carries
attribution requirements. OSM gets you a walkable city for nothing, and in
Amsterdam it is genuinely excellent data — see below.

## How it works

**Projection.** The first location you pick becomes the origin. Everything after
that is a local east/north plane in metres, which is accurate to well under a
centimetre across the few kilometres you can actually walk.

**Buildings.** Footprints are extruded by hand rather than with `ExtrudeGeometry`,
so each wall quad gets a UV mapped to real metres — one texture repeat is 6 m of
street frontage by 6.6 m of height. That is what makes the window rows line up at
a believable size instead of stretching to fit whatever the footprint happens to
be. Roofs are triangulated with holes, so courtyards stay open. Heights come from
the `height` tag, else `building:levels × 3.25`, else a guess.

Amsterdam is close to the ideal test case: in a sample tile of the Jordaan, 1289
of 1470 buildings carry a surveyed height, because the Dutch BAG/3DBAG cadastre
has been imported into OSM. Median building height 13.5 m, median footprint 78 m²
— narrow canal houses, exactly right.

**Canals.** Water sits 1.9 m below street level behind a stone quay wall, which is
what actually makes a canal read as a canal rather than blue paint. That means the
ground cannot be one big plane, or it would cover the water; instead every tile
draws its own land quad with the canal polygons clipped to the tile square and
punched out as holes.

**Bridges.** Ways tagged `bridge` are subdivided and arched on a sine profile, get
a deck with visible sides, and are registered as walkable surfaces — so the camera
rides up over the span and back down the other side.

**The sun is real.** Rather than a few canned lighting presets, the low-precision
NOAA solar position algorithm places the sun from the actual latitude, longitude
and moment — by default, right now, wherever you are standing. It is accurate to
a hundredth of a degree at the solstices, and it gets the southern hemisphere
right: in Sydney in January the sun is correctly in the *north*. Sky gradient,
fog, sun colour and bounce light all interpolate off solar altitude, and the
windows light up once it drops below the horizon. Scrub the hour with
<kbd>[</kbd> / <kbd>]</kbd> and you can watch the shadows swing round the street.

The clock beside it is real civil time in that city. The sun itself only needs a
UTC instant, so it is correct either way, but showing the right *wall* time needs
the actual zone — that comes from a one-shot lookup on `timeapi.io`, after which
`Intl` handles DST and half-hour offsets exactly. If the lookup fails the clock
falls back to longitude, which stays solar-sane but drifts from the wall clock by
the DST offset.

**Streaming, politely.** The world is 600 m tiles, and the grid is pinned to the
globe rather than to wherever you searched from — so two visits to the same
street ask for the same cells whatever address you arrived by, and the second
visit is free. Every tile is cached in IndexedDB for a fortnight, so reloads cost
nothing at all.

Tiles are fetched only when you can actually see into them: one while you are
mid-cell, never more than four, instead of firing the whole surrounding nine the
moment you land. One request is in flight at a time with a real pause between
them, and Overpass pushing back triggers an exponential retreat rather than a
retry storm. This matters — Overpass does not politely throttle a noisy client,
it firewalls the IP, and a refused TCP connection surfaces in the browser as a
bare `Failed to fetch`. If you see that, you are blocked rather than offline; it
clears on its own.

Ways are deduplicated by OSM id across tiles, and tiles more than two cells away
are disposed of, so a long wander does not grow without bound.

**Collision.** Building footprints go into a 24 m spatial grid; movement is tested
against the polygons in the neighbouring cells and slides along walls rather than
stopping dead.

**View arms.** Two arms parented to the camera, swinging on the same phase as
the head bob so they keep step with your actual pace, sliding out of frame when
you stand still. Toggle them off on the start page.

Two things are worth knowing if you ever move them. The upper arm is deliberately
not drawn: the shoulder sits behind your eyes, so a mesh there puts geometry
across the near plane, and geometry straddling the near plane smears across the
entire screen — the giveaway is a positive camera-space z on the arm's bounding
box. The pivot still drives the forearm; you simply never see your own upper arm
in first person. And the whole pose is solved rather than eyeballed: the shoulder
angles are chosen so both hands stay framed through the full swing while the
elbow stays below the viewport, which is worth re-checking if you change the
field of view.

Hand roll is a single expression, `-side * PI * (1 + f) / 2`, where `f` is how
far into the gesture you are. A quarter turn gives palms inward with the thumbs
riding on top, which is how hands sit when you run; a half turn presents the
palm to the camera.

**People.** Pedestrians walk the road graph, sometimes alone and sometimes in
twos and threes keeping pace together. They are drawn as six `InstancedMesh`
objects — head, torso, two legs, two arms — so the whole crowd is six draw calls
however many are out, with per-instance colour for clothing and skin. Each one
follows a way at a sidewalk offset, takes another way at each junction (OSM ways
share endpoint nodes, so the junctions fall out of the data for free — 147 of
them in one Jordaan tile), and turns on its heel at a dead end. They ride bridge
decks like you do.

They also mind being flipped off. Hold <kbd>F</kbd> where someone can see it —
within 14 m, in front of you, and not facing away — and their face reddens. Do it
to the same person twice and they abandon their errand and come after you,
leaning in, pumping their arms and holding station about 1.4 m off your shoulder.
They manage 2.3–2.65 m/s against your 6 m/s run, so you can walk into trouble but
you can always outrun it. Offence is taken on the rising edge of the gesture, so
holding the key does not stack.

They are spawned and retired in a ring around you, drawn from a shortlist of
nearby segments that is refreshed as you move — sampling uniformly from every
loaded road mostly misses, because the tiles cover well over a square kilometre.
Population is topped up in a burst rather than one group per frame, so the crowd
size does not depend on how fast your machine renders. Toggle them off on the
start page.

## Textures, and the street-view question

The facades are procedurally drawn to a canvas — plaster grain, storey bands,
window reveals, glazing bars — and UV-mapped in real metres so the window rhythm
is 3 m regardless of building size. Open alternatives, in order of how much they
buy you:

- **CC0 material libraries** — [ambientCG](https://ambientcg.com/) and
  [Poly Haven](https://polyhaven.com/textures) publish photogrammetry-derived PBR
  brick, plaster, roof and paving sets under CC0. Public JSON APIs, CORS open, no
  attribution required. A curated dozen keyed off region would be the single
  biggest visual upgrade here, with no coverage gaps and no licence friction.
- **OSM's own appearance tags** — `building:colour`, `building:material`,
  `roof:colour`, `roof:shape`. Free and already in the Overpass response, but the
  coverage is thin: in the Jordaan sample tile, 87.4% of buildings carry a
  surveyed *height* and only 0.1% carry a colour. Worth honouring where present,
  but it will not carry a city.
- **[KartaView](https://kartaview.org/)** — open street-level photography,
  CC BY-SA. Genuinely frictionless: no token at all, `Access-Control-Allow-Origin: *`,
  and it returns geotagged JPEGs with a compass heading. There is a 2021 shot
  three metres from the Prinsengracht spawn point.
- **[Mapillary](https://www.mapillary.com/)** — much larger coverage, same
  CC BY-SA licence, but needs a free OAuth token.

Projecting that photography onto the walls is the obvious next thought, and it is
harder than it looks. The imagery is shot from car dashboards: the lower third is
windscreen and bonnet, and the facades you want are behind bikes, trees, bollards
and pedestrians. Doing it properly means multi-view fusion plus semantic
segmentation to mask out the clutter, then relighting to reconcile exposures —
which is why the pipelines that do this well are large. A far cheaper use of the
same data is to show the nearest real photo in a corner panel as you walk, so you
can hold the model up against the street it came from.

Google Street View imagery is not an option regardless — using it this way is
against their terms.

## Known limits

- **The ground is flat.** There is no terrain elevation, so hilly cities are
  wrong. Amsterdam does not care. Fixing this means sampling a DEM — Terrarium
  tiles on AWS are free and CORS-friendly — and displacing everything by it.
- **Roofs are flat.** No gables, spires or domes; OSM's `roof:shape` is ignored.
- **You can walk over water.** Canals are not solid, because making them solid
  would strand you whenever a bridge is missing from the data. You float across.
- **Interiors do not exist.** These are hollow shells with no doors.
- **Pedestrians do not avoid anything** — not you, not each other, not walls.
  They follow their way and walk through whatever is in it, and a pursuer will
  come at you straight through a building rather than round it.
- **Landmark heights are guessed** when OSM has none, so the odd church tower is
  the wrong size.
- **Overpass is a shared free service** run on donated hardware. Tiles are cached
  and paced to stay well inside its limits, but it is still someone else's
  machine. Two of the three configured mirrors resolve to the same host, so the
  failover is thinner than it looks.

## Poking at it

Everything is exposed on one console handle:

```js
city.player           // { x, z, y, yaw, pitch }
city.tiles            // loaded 600 m blocks
city.go(52.37, 4.89)  // teleport
city.solarPosition(new Date(), 52.37, 4.89)   // { alt, az } in radians
city.bridges          // walkable decks
```

## Licence and attribution

Map data © OpenStreetMap contributors, [ODbL](https://www.openstreetmap.org/copyright)
— the attribution shown in the corner needs to stay if you publish this anywhere.
Geocoding by Nominatim, subject to its
[usage policy](https://operations.osmfoundation.org/policies/nominatim/).
