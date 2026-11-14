# Starter notes for a single-file three.js app

These notes come from building a 3D apartment layout planner: a floor plan traced into a 3D shell, furniture you can place, move and resize, a first-person walk mode, wall paint, and a built-in Claude assistant that makes furniture from a chat. The app is one `index.html` of about 3,800 lines with no build step, hosted on GitHub Pages.

Copy this file into your own project as `CLAUDE.md` and delete whatever does not apply. Everything here was learned the hard way and worked in practice.

## Shape of the app

- One `index.html`: markup, CSS and a single inline script. No bundler, no framework, no npm at runtime. It deploys by pushing to a branch that GitHub Pages serves.
- three.js r128 from cdnjs (`three.min.js`, global `THREE`). Pin an exact version. Later versions moved to ES modules only, which breaks the plain script tag approach.
- The script is split into sections by banner comments (`// ================= name`), so a reader or Claude can find their way around a long file: language, renderer, scene shell, object bank, items, selection, camera, walk mode, UI panels, persistence, assistant, frame loop, start.
- Large images (photos used as textures) are embedded as base64 data URIs, each on its own very long line. Never print those lines. Search with `grep -n ... | cut -c1-300`, and edit with small scripted replacements that assert the anchor text occurs exactly once.
- A `noindex` robots meta tag keeps a personal tool out of search results.

## Coordinates and units

- World units are metres. x runs east, z runs south, y is up. Pick one convention on day one and write it down.
- Object rotation is stored in degrees and applied as `group.rotation.y = -r * PI / 180`, so a positive angle turns clockwise on the plan. Local +z is the front of an object.
- Tracing a floor plan image: measure the scale from a known dimension on the plan (mm per pixel) and pick an origin pixel. Then `X(px) = (px - ox) * S` and `Z(py) = (py - oy) * S`. Shell helpers take plan pixels, so walls can be read straight off the image. Check the traced area against the stated area. Within 1 to 2% is good, and the rest is plan inaccuracy.
- Ceiling and door heights usually are not on a plan. Make them named constants and note that they are assumptions.

## Objects

- An item is plain data: `{id, k, x, z, r, w, d, h, y, lock, top}` (type key, centre, rotation, size, height off the floor, locked, sits on another object). Its `THREE.Group` hangs off it at runtime and is never saved.
- Every type has a builder `BUILD[name](g, w, d, h)` that fills the box `-w/2..w/2`, `0..h`, `-d/2..d/2` exactly, at real-world size. Two helpers cover most needs: `part(g, x0, z0, x1, z1, y0, y1, mat)` for a box and `tube(g, x, z, rx, rz, y0, y1, mat)` for a cylinder. Builders should take the size as given rather than scale a fixed model, so resizing keeps legs and panel thicknesses right.
- A `BANK` array lists the types: `{k, b, n: [label in each language], w, d, h, y?}`. Category lists drive the filters in the picker.
- For curved or angled pieces (an L or horseshoe sofa), chain straight sections: each has a length, a depth and a turn from the previous one. Build them in a local group, measure its bounding box, then scale the result to the item's `w x d x h`. Fit the section values so the natural bounding box is already close to the real size, or the scaling will stretch the piece.
- To match a maker's plan drawing, segment the drawing into a mask (pixels darker than the background), rasterise your sections the same way, and search the section values for the best overlap. It takes a few seconds and beats guessing.

## Performance

- A scene like this is limited by draw calls, not triangles. Merge meshes that share a material (and shadow settings) into one mesh, once for the static shell and once per object after it is built. Builders can still create hundreds of small parts. In this app that took about 1,640 draw calls a frame down to about 210.
- Do not redraw shadows every frame: set `renderer.shadowMap.autoUpdate = false` and keep a `shadowDirty` flag that every change to the scene sets. Turning the camera does not need new shadows. Redraw every frame only while things animate.
- Render on demand: a `dirty` flag, set by input and changes, so an idle page does not burn a phone battery.
- Canvas textures (wood grain, fabric, marble, wallpaper) are cheap and look good. Seed the noise (`rng(seed)`) so they come out the same on every load. Paint expensive ones once at load and make them tile seamlessly. For materials that must keep a real-world scale on boxes of any size, compute planar UVs from the world size of each face instead of stretching 0..1 across it.

## Interaction

- Phone first: every keyboard shortcut needs a touch equivalent. Check a 390 px wide viewport as well as desktop.
- Dragging an object on the floor: raycast against a horizontal plane at floor height, not against the object. Show live gap measurements to the nearest walls while moving, and only then. Constant labels are clutter.
- A wall magnet (snap the object's back to a wall within a few centimetres) helps a lot, but it is not a barrier. Say so.
- Objects that sit on others (books on a shelf, a lamp on a table): on commit, find the surfaces under the object's centre and settle on the highest one below a cap. When a support moves, record its riders in the support's own frame and put them back afterwards.
- Undo as a stack of serialised snapshots is simple and good enough. Group everything one user action (or one assistant request) changes into one step.
- Fly camera for desktop, in the style of a game editor: hold the right mouse button to look around, W A S D and E / Q to move, the wheel for speed while held. Capture those keys only while flying so they do not trigger other shortcuts.

## Walk mode

- A first-person camera at a fixed eye height, with collision against walls and objects using short horizontal raycasts.
- Hide UI chrome while walking and offer a small drawer instead.
- Route finding for an animated character (a pet, for example): sample the floor into 15 cm cells when walk mode starts, a few hundred cells per frame so it does not stall, then run A*. Detect obstacles by casting rays upward from below the floor. Casting downward from inside a wall finds nothing, which was a real bug.

## Stored state

- localStorage, versioned: `{v, items}`. When a change must alter layouts people already have, bump `v` and add a one-off migration that runs on load. Keep migrations small and idempotent.
- A built-in default layout for first-time visitors. When the owner sends a new default, paste it in unchanged.
- Wrap every storage read and write in try/catch. Private windows and blocked storage throw.

## Built-in Claude assistant (bring your own key)

- The page calls the Messages API directly from the browser with the header `anthropic-dangerous-direct-browser-access: true`, using the viewer's own API key.
- The key lives only in that browser's localStorage. Never put a key in the page, the repository or a chat. Hide the assistant tab until a key is saved, and reveal the key form with a URL hash (for example `#claude`) so ordinary visitors never see it.
- Give the model tools that act on the app (`place_item`, `update_item`, `remove_items`, `define_object`, and so on) with strict JSON schemas. Return useful results: clearances to each wall and overlaps with other objects (a separating axis test on the plan is enough), so the model can correct itself.
- Put the long, stable context (system prompt, scene geometry) first and cache it. Start each user turn with the current state.
- Let the model define new object types as data: parts that are boxes, cylinders or spheres, each with min and max on every axis in the object's own frame, plus colour, finish and texture. Sanitise every spec before building it (clip sizes, counts and values). Give specs a revision number so a built-in copy can override older local copies.
- If an optional feature (a beta header, strict schemas) is rejected by the API, drop it automatically and retry, so the app keeps working.

## Internationalisation

- Keep every user-facing string in a `TXT[lang]` table, look it up with `tr(key)`, and mark static HTML with `data-i18n`. Re-render on a language change via registered callbacks. Adding strings for every language at the time you write them is far cheaper than retrofitting.

## Testing without a test suite

- Load the page in headless Chromium through Playwright with `--use-gl=swiftshader --enable-unsafe-swiftshader --ignore-gpu-blocklist`. Collect `pageerror` events and take screenshots, then actually look at them.
- Test from a copy of the page with the CDN URL swapped for a local `three.min.js` of the same version, in case the network is restricted.
- To reach internals, inject a line before the start of the main script that exposes what you need on `window` (an item list, a camera setter, a report function). Keep those hooks out of the committed file.
- Drive state through those hooks. Dragging at guessed pixel positions is unreliable.
- For the assistant, intercept the API URL with `page.route` and answer with scripted `tool_use` responses. The intercepted requests show exactly what would be sent.
- For anything animated, step the simulation with fake timestamps in a loop, count states, then check a still frame.
- Start every check from an empty browser profile, so defaults and migrations are tested the way a new visitor sees them.

## Working conventions that paid off

- Build objects to the real dimensions the owner gives, and state any dimension you had to assume.
- Keep a "Known limits" list in this file and add to it honestly.
- Keep this file current as the app changes. A stale guide is worse than none.
