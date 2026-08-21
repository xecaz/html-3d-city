# HTML City

Type an address. Land in it. Walk away down the street.

A single HTML file that pulls a real city out of OpenStreetMap and rebuilds it as
walkable 3D geometry in the browser. Search `Prinsengracht 263, Amsterdam`, press
Enter, and a moment later you are standing on the quay in first person with the
arrow keys under your fingers. Keep walking and the next blocks are fetched and
built in front of you, so there is no edge of the world to bump into.

No build step, no bundler, no `node_modules`, no API keys, no account. One file,
about a thousand lines, and three.js from a CDN.

## Run it

```bash
python3 -m http.server 8777
# then open http://127.0.0.1:8777/index.html
```

Any static server will do — it must be `http://`, not `file://`, or the module
import and the API calls will be blocked. Jump straight to a spot with a hash:

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

The search box takes a street address, a place name, or a raw `lat, lon` pair.

## Where the data comes from

Everything is public and free, and nothing is scraped.

- **[OpenStreetMap](https://www.openstreetmap.org/copyright)** via the **Overpass
  API** — building footprints, heights, roads, canals, bridges, parks. Requested
  per tile as you move, from three mirrors with automatic failover.
- **[Nominatim](https://nominatim.openstreetmap.org/)** — turns what you type into
  a latitude and longitude.
- **[three.js](https://threejs.org/) r169** — WebGL rendering, loaded from a CDN
  through an import map.

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

**Streaming.** The world is 600 m tiles. The one you land in loads first, the eight
around it follow in the background, and more are queued as you walk. Ways are
deduplicated by OSM id across tiles, and tiles more than two away are disposed of,
so wandering for a long time does not grow without bound.

**Collision.** Building footprints go into a 24 m spatial grid; movement is tested
against the polygons in the neighbouring cells and slides along walls rather than
stopping dead.

## Known limits

- **The ground is flat.** There is no terrain elevation, so hilly cities are
  wrong. Amsterdam does not care. Fixing this means sampling a DEM — Terrarium
  tiles on AWS are free and CORS-friendly — and displacing everything by it.
- **Roofs are flat.** No gables, spires or domes; OSM's `roof:shape` is ignored.
- **You can walk over water.** Canals are not solid, because making them solid
  would strand you whenever a bridge is missing from the data. You float across.
- **Interiors do not exist.** These are hollow shells with no doors.
- **Landmark heights are guessed** when OSM has none, so the odd church tower is
  the wrong size.
- **Overpass is a shared free service.** It rate-limits. If a tile fails to load
  it is retried against another mirror; be decent about how hard you hammer it.

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
