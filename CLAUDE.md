# Capston N11.01 layout planner

A single-page 3D layout planner for flat N11.01 at The Capston (Embassy Gardens, Nine Elms, London). Owner: Giray, with his wife Teodora. They are buying N11.01 and the adjacent N11.02 and use the planner to try furniture arrangements before completion. It was built in a Claude Cowork session and moved here to be hosted on GitHub Pages.

## Working with Giray

- Call him Giray. He is a software engineer by background.
- Never use em dashes in replies to him. Use commas, hyphens or new sentences.
- He often works from his phone, so anything keyboard-only needs a touch equivalent.
- He adds furniture by describing it in chat, usually with a photo and exact dimensions. Build it to those real-world dimensions.
- His mother views the planner in Turkish, so every user-facing string needs an English and a Turkish version.

## The repository

- `index.html` is the whole app: markup, CSS and one inline script, about 1,650 lines. No build step.
- three.js r128 is loaded from cdnjs (`three.min.js`, global `THREE`). Fonts are Marcellus and Figtree from Google Fonts.
- Photos the owner supplied (two rugs, the TV picture, the Neva chair linen) are embedded as base64 data URIs on single very long lines. Do not print those lines. Search with `grep -n ... | cut -c1-300`, and edit with small scripted replacements that assert the anchor text occurs exactly once.
- Deploy: GitHub Pages, "Deploy from a branch", `main`, root. There is a `noindex` robots meta tag. The repo and site are public on a free plan, so do not add personal details beyond what is already in the file.

## Coordinates and scale

- World units are metres. x runs east, z runs south, y is up. The flat's north wall is at the top of the plan.
- The shell was traced from a floor plan screenshot. `S = 0.01642` metres per plan pixel, `X(px) = (px - 304) * S`, `Z(py) = (py - 956) * S`. Shell helpers (`box`, `slant`, `wall`, `header`, `rbx`) take plan pixels.
- The window wall is slanted: glazing line `gx(py) = 702 + 0.342 * (py - 998)`, facade angle 71.1 degrees. The living room corner at the window is 109 degrees.
- Ceiling height `H = 2.6` is an assumption, as are door heights of 2.1. Wall lengths are estimates from the plan (traced area 87.0 sq m against a stated 88.4), so treat every dimension of the shell as approximate.

## Code map (section comments in the script)

- `language`: `TXT.en` / `TXT.tr`, `tr(key)`, `data-i18n` attributes, `onLang` callbacks. A URL ending in `#tr` opens in Turkish.
- `plan coordinates`, `textures`, `materials`: `M(color, roughness)` makes a material from an sRGB hex. Canvas textures use `rng(seed)` for repeatable noise.
- `the fixed shell`: walls, glazing, kitchen, bathrooms, built-in joinery. Groups: `walls`, `tall`, `low`. Each built-in is registered under `curFix` so it can be selected and measured but not moved. Cut-away mode scales `walls` and `tall` to 0.42 height.
- `furniture bank`: `BUILD[type](g, w, d, h)` builders, the `BANK` array, `BK` lookup, `CATS` category lists, `START` (the default layout).
- `items`: the movable objects. `norm`, `plain`, `rebuild`, `pose`, `setItems`, `commit`.
- `selection, gizmo`, `magnetism`, `sits on furniture`, `inspector`, `keyboard shortcuts`.
- `camera rig`, `walk mode`, `dock`, `layouts`, `light, frame loop`.
- `Nemo`: the dog, described below.

## Objects

An item is `{id, k, x, z, r, w, d, h, y, lock, top, g}`: bank key, centre position, rotation in degrees, size, height off the floor, locked flag, "sits on furniture" flag, and its `THREE.Group`. `g.rotation.y = -r` degrees. Local +z is the front of an object.

To add a furniture type:

1. Write a builder `BUILD.myThing = function (g, w, d, h) { ... }` that fills the box from `-w/2..w/2`, `0..h`, `-d/2..d/2` exactly. Helpers: `part(g, x0, z0, x1, z1, y0, y1, mat, noShadow)` for a box, `tube(g, x, z, rx, rz, y0, y1, mat)` for a cylinder.
2. Add a `BANK` entry `{ k, b, n: [english, turkish], w, d, h, y?, top? }`. `k` is the unique key, `b` the builder name.
3. Add the key to a list in `CATS` so it shows under the right filter. A new category also needs `c<name>` strings in both languages.

Conventions already in use: wall-hung pieces carry a default `y` (wall shelving 0.41, art centred at 1.45). Desk-top pieces use `top: true`. Rugs are under 6 cm tall, which is how Nemo's route finding knows to ignore them.

"Sits on furniture" (`top`): on every `commit()` the object settles onto a surface under its centre. It keeps its current shelf if that shelf is still under it, otherwise it takes the highest surface up to 1.45 m. `surfacesUnder` lists the levels and `stepLevel` backs the Shelf up / Shelf down buttons. Moving a support carries its riders (`ridersOf`).

## Stored state

Browser localStorage, all per device:

- `capston-n1101-work`: the working layout, `{v, items}`. `v` is currently 4. Loading an older version runs one-off migrations (rug sizes, first fill of shelving with books). Bump `v` and add a migration when a change must alter layouts people already have.
- `capston-n1101-saved`: named layouts.
- `capston-lang`, `capston-magnet`, `capston-screens` (glazed partitions, hidden unless `on`), `capston-nemo`.

The layouts section first looks for `window.claude.use('db')`, which only exists when the page is hosted as a Claude artifact. On GitHub Pages that is absent and named layouts fall back to localStorage, so they are not shared between visitors. Everyone still opens on `START`.

## Walk mode and Nemo

Walk mode is a first-person camera with collision (`freeWay`), a ceiling, full-height walls and a painted backdrop outside the windows. All of those are hidden outside walk mode.

Nemo is the owners' dog, a 10 kg apricot labradoodle, modelled in a block style. He exists only in walk mode, is never saved, and can be toggled off.

- State machine in `think`: `wander`, `follow`, `greet`, `rest`, `toperch`, `jump`, `perched`. `decide` picks the next activity. About 11% of decisions go to his bed when one is placed (`dogBed`), which works out near 20% of his time.
- Commands: Space or the Call button brings him to the player. B or the Bed button sends him to the nearest bed.
- Route finding: the floor is sampled into 15 cm cells when walk mode starts (`gridBuild`, a few hundred cells per frame), then `plan` runs A*. Obstacles are detected by casting rays upward from below the floor (`obstacleAt`). Casting downward from inside a wall finds nothing, which was a real bug.
- `moveToward` follows the route and uses short horizontal rays (`blocked`) to slide around anything the grid missed.

## Testing

There is no test suite. The method that worked: load the page in headless Chromium through Playwright with `--use-gl=swiftshader --enable-unsafe-swiftshader --ignore-gpu-blocklist`, collect `pageerror` events, and take screenshots.

- Use a copy of the page with the cdnjs URL swapped for a local `three@0.128.0` if the network is restricted.
- To reach internals, inject a line just before the `// ================= start` comment that exposes what you need on `window` (for example a camera setter, `nemo`, or `items`). Keep those hooks out of the committed file.
- Dragging by guessed pixel positions is unreliable. Drive state through injected hooks instead.
- Keyboard shortcuts are ignored while an input has focus.
- For Nemo, step the simulation directly with fake timestamps (`nemo.step(ts)` in a loop) and count states, then look at a still frame. Check phone width (390 px) as well as desktop.

## Known limits

- Furniture is schematic. Heights of anything not specified by the owner are assumptions.
- The magnet snaps objects to walls but is not a barrier. Objects can still overlap.
- Sitting objects use their centre point only, and do not stack on each other.
- Wall art floats above the walls in cut-away mode, because those walls stop at about 1.1 m.
- Resizing a textured rug or framed picture stretches the image.
- The window view is an invented painting, not the real outlook from the flat.
- Nemo occasionally fails to reach the player when called while he is up on furniture in a tight spot. Calling again works.

## Ideas discussed but not built

- A scripted walk-through video rendered frame by frame.
- Opening on a chosen saved layout by default for first-time visitors.
- The same planner for N11.02, and for the two flats joined.
