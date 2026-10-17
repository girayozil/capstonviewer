# Capston N11.01 layout planner

A single-page 3D layout planner for flats N11.01 and N11.02 at The Capston (Embassy Gardens, Nine Elms, London). Owner: Giray, with his wife Teodora. They are buying both neighbouring flats and use the planner to try furniture arrangements before completion. The title in the header is a picker that switches between them, or shows both together with the shared hallway. It was built in a Claude Cowork session and moved here to be hosted on GitHub Pages.

## Working with Giray

- Call him Giray. He is a software engineer by background.
- Never use em dashes in replies to him. Use commas, hyphens or new sentences.
- He often works from his phone, so anything keyboard-only needs a touch equivalent.
- He adds furniture by describing it in chat, usually with a photo and exact dimensions. Build it to those real-world dimensions.
- His mother views the planner in Turkish, so every user-facing string needs an English and a Turkish version.

## The repository

- `index.html` is the whole app: markup, CSS and one inline script, about 2,800 lines. No build step.
- three.js r128 is loaded from cdnjs (`three.min.js`, global `THREE`). Fonts are Marcellus and Figtree from Google Fonts.
- Photos the owner supplied (two rugs, the TV picture, the Neva chair linen) are embedded as base64 data URIs on single very long lines. Do not print those lines. Search with `grep -n ... | cut -c1-300`, and edit with small scripted replacements that assert the anchor text occurs exactly once.
- Deploy: GitHub Pages, "Deploy from a branch", `main`, root. There is a `noindex` robots meta tag. The repo and site are public on a free plan, so do not add personal details beyond what is already in the file.

## Two flats

- `FLATS` at the top of the script holds what differs per flat: plan scale `S` and origin (`ox`, `oy`), the facade line `g0`, camera `views`, walk start, drag and add limits, Nemo's roaming box and grid, the sun target, and the labels of the last two view buttons.
- `FLAT` is chosen by `?flat=n1102` in the address, else the last flat picked on this device (`capston-flat`), else N11.01. The picker reloads the page with `?flat=`; only the chosen flat is built.
- The shell of each flat is its own function, `shellN1101()` and `shellN1102()`. N11.01 returns where its painted window view sits (one plane per outlook). N11.02 returns `{ pano: [px, py] }`: `panoN1102` paints a dusk panorama after the owner's balcony photos (red brick tower, glass office and park to the left for the kitchen window; the embassy cube, road and the Thames towards Battersea to the right for the living room) on a curved band 90 m out, laid out by plan bearing (kitchen window 71 degrees, south windows 180). `KN` maps painted bearings onto the band, so features can be slid round without repainting. The right side (from the embassy on) is drawn directly in band bearings with `bz`, shifted 8 degrees right: Nine Elms Lane, west of the building (owner's site diagram), comes in from the lower right and runs to Battersea Power Station at the vanishing point `VP` (band bearing about 186), the Thames beyond it further right with its near bank `NEAR`, and the chimneys are drawn taller than life so they read.
- Stored layouts are per flat: `capston-<flat>-work` and `capston-<flat>-saved`; the shared artifact store uses `layouts` for N11.01 and `layouts-n1102` for N11.02. `START` (the default layout) applies to N11.01; `START02` is N11.02's (the master bedroom, the studio desk, and the living room with the Nockeby, Layered rug, wing chair, sage table, TV, Teodora's painting, the dog crate from `SPECS`, and two HAY Palissade chairs on the balcony, `BALC02`), shifted with `shift02` in the combined view. N11.02 comes with a built-in dining bench (`diningBench`, fixture `bench`, cream boucle) filling the inlet east of its kitchen island to the end of the marble bar, which rests on the bench's back (no separate leg). N11.02's rooms are the Studio (`bed1`, Teodora's, south-west) and the Master bedroom (`bed2`, north).
- The combined view (`?flat=both`, `FLATS.both`) builds both shells in N11.01's frame: `setFrame(flat, dx, dz)` moves the plan coordinates, and N11.02 sits at `DX2, DZ2` so that N11.01's kitchen wall is N11.02's north wall and the slanted facades are one line. Fixture keys get a suffix per flat (`door@1`, `door@2`; `fixBase` strips it for names). `hallway()` adds the shared corridor between the front doors (floor, lift and stair core, the neighbour's door). Floors and ceilings are groups (`floorMesh`, `ceiling`) holding every piece; `POLYS` keeps each flat's outline in world metres.
- Doors (combined view only): `hingeDoor` puts each front door on a hinge swinging into its flat. In walk mode the door in front of you within 1.8 m (`nearDoor`) shows a button on it (`#doorbtn`) and opens or closes with it or E; doors swing in about 0.6 s and Nemo's route grid is rebuilt.
- Furniture in the combined view comes from both flats' own working layouts (N11.02's moved by `DX2, DZ2`); `commit` saves each piece back to the flat it stands in (`inN1102`). Named layouts in this view are kept under `capston-both-saved`.
- N11.02 was traced from the brochure plan at 20.4 mm per pixel; its six stated room sizes match within 1.2% when measured wall to window line. Its internal area of 98.5 sq m, the 5.7 sq m balcony and the facade angle (the same 71.1 degrees as N11.01) come from the brochure.

## Performance

- The scene is limited by draw calls, not triangles. `mergeUnder(root)` merges meshes that share a material, fixture key and shadow settings into one mesh: the shell after it is built (`mergeShell`, which then rebuilds `FIX` from the merged meshes) and each furniture item's body in `rebuild`. Builders can still make as many small parts as they like. Before this, N11.01 drew about 1,640 calls a frame; after, about 210.
- Shadows are not redrawn every frame (`renderer.shadowMap.autoUpdate = false`). Set `shadowDirty = true` whenever something in the scene moves (`pose`, `changed`, `commit`, `rebuild`, `setWalls`, `setScreens` already do); turning the camera does not need it. Walk mode redraws shadows every frame because Nemo and the doors move.

## Coordinates and scale

- World units are metres. x runs east, z runs south, y is up. The flat's north wall is at the top of the plan.
- Each shell was traced from a floor plan image. `X(px) = (px - FL.ox) * S`, `Z(py) = (py - FL.oy) * S`: N11.01 uses 16.42 mm per pixel from (304, 956), N11.02 20.4 mm per pixel from (259, 349). Shell helpers (`box`, `slant`, `wall`, `header`, `rbx`, `slantGlass`, `flatGlass`, `floorShape`) take plan pixels.
- The window wall is slanted: glazing line `gx(py) = g0[0] + 0.342 * (py - g0[1])`, facade angle 71.1 degrees in both flats. The living room corner at the window is 109 degrees.
- Ceiling height `H = 2.6` is an assumption, as are door heights of 2.1. Wall lengths are estimates from the plan (traced area 87.0 sq m against a stated 88.4), so treat every dimension of the shell as approximate.

## Code map (section comments in the script)

- `language`: `TXT.en` / `TXT.tr`, `tr(key)`, `data-i18n` attributes, `onLang` callbacks. A URL ending in `#tr` opens in Turkish.
- `plan coordinates`, `textures`, `materials`: `M(color, roughness)` makes a material from an sRGB hex. Canvas textures use `rng(seed)` for repeatable noise. The marble (`marbleTex`, Calacatta Viola style breccia after the owner's photo) is painted per pixel once at load, seamless, one tile per 2.4 x 1.2 m; `box` and `rbx` call `planarUV` for any material with `userData.planar = [tile w, tile h]` (the marble, and the pippy oak of the guest wardrobe from `pippyTex`) so it maps at real scale instead of stretching per face. The guest wardrobe is built by `pippyWardrobe`. The floor (`parquetTex`) is the brochure's checkerboard parquet: squares of five mid-brown oak strips with grain, one tile per 0.648 m.
- `the fixed shell`: walls, glazing, kitchen, bathrooms, built-in joinery. Groups: `walls`, `tall`, `low`. Each built-in is registered under `curFix` so it can be selected and measured but not moved. Cut-away mode scales `walls` and `tall` to 0.42 height. The utility cupboard and hall closet use `builtIn(x0, y0, x1, y1, face, n)`: a white body and bulkhead that read as wall, `n` flush oak doors with shadow gaps and slim black bar handles. `pippyWardrobe(x0, y0, x1, y1, n, face)` builds the inset pippy oak wardrobes. Both work on any face (`'n'`, `'s'`, `'e'`, `'w'`) through `faceMap`.
- `furniture bank`: `BUILD[type](g, w, d, h)` builders, the `BANK` array, `BK` lookup, `CATS` category lists, `START` (the default layout).
- `items`: the movable objects. `norm`, `plain`, `rebuild`, `pose`, `setItems`, `commit`.
- `selection, gizmo`, `magnetism`, `sits on furniture`, `inspector`, `keyboard shortcuts`.
- `camera rig`, `walk mode`, `dock`, `layouts`, `light, frame loop`.

## Where the controls live

- Header: title and one EN / TR toggle (`#langbtn`).
- On the 3D view: Walk through (top right), Measure and the Display menu below it (`#tools`; `#dispmenu`: Walls, Partitions and Measurements), the view buttons along the bottom (`#views`; Alt+J toggles Plan), and a floating Undo (`#undo`) shown only when there is something to undo. The hint shows on a first visit only.
- Item panel (`#inspector`): modes Move / Rotate / Resize as a segmented control plus Magnet; actions Duplicate / Lock / Delete; less common ones (Sits on furniture, Fill with books, Shelf up / down) behind More (`#insp-more`).
- Dock (`#dock`): Furniture and Layouts tabs, `openTab(name)`. On phones it sits under the 3D view, both tabs can be closed, and `body.editing` hides it while an item is selected. On screens 900 px and wider `placeBank` moves the whole dock into the right sidebar (`#side`), with one tab always open.
- Wall gap measurements (`gap0..3`, `gapLines`) show only while an item is being moved, turned or resized (`measuring`); a selected item shows just its own size. The Measurements toggle (`dimsOn`, stored as `capston-dims`) hides both.
- Flying (mouse only, Unreal Editor style, `fly`): hold the right mouse button to look around from where the camera stands, W A S D to fly along the view, E / Q up and down, the wheel for speed (`capston-flyspeed`, 0.3 to 30 m/s) while the button is held. `startFly` shortens the orbit distance to at most 4 m without moving the camera, so letting go leaves orbit and zoom working from the new spot. Looking above the horizon sets `cam.free`, which lifts the orbit's 1.5 rad limit until the next left drag. While flying those keys are captured, so they do not switch tools. Shift-drag pans (right-drag used to).
- The ruler (`ruler`, Measure button or M, Esc to leave): tap two points on any visible surface for the straight distance, drawn in orange with `#meastag`. A second point nearly in line with the first along x, z or height snaps into line; a sloping line also shows its level and upright parts. A drag still turns the view, and a third tap starts again. It switches off in walk mode.
- `Nemo`: the dog, described below.

## Objects

An item is `{id, k, x, z, r, w, d, h, y, lock, top, g}`: bank key, centre position, rotation in degrees, size, height off the floor, locked flag, "sits on furniture" flag, and its `THREE.Group`. `g.rotation.y = -r` degrees. Local +z is the front of an object.

To add a furniture type:

1. Write a builder `BUILD.myThing = function (g, w, d, h) { ... }` that fills the box from `-w/2..w/2`, `0..h`, `-d/2..d/2` exactly. Helpers: `part(g, x0, z0, x1, z1, y0, y1, mat, noShadow)` for a box, `tube(g, x, z, rx, rz, y0, y1, mat)` for a cylinder.
2. Add a `BANK` entry `{ k, b, n: [english, turkish], w, d, h, y?, top? }`. `k` is the unique key, `b` the builder name.
3. Add the key to a list in `CATS` so it shows under the right filter. A new category also needs `c<name>` strings in both languages.

Conventions already in use: wall-hung pieces carry a default `y` (wall shelving 0.41, art centred at 1.45). Desk-top pieces use `top: true`. Shelving pieces (`isShelving`) arrive empty. "Fill with books" in the item panel runs `fillShelving` (for the slatted bookcase, the books and plants in `SLATFILL`, as separate sitting items); once anything sits on the piece the same button reads "Clear shelves" and removes it. Rugs are under 6 cm tall, which is how Nemo's route finding knows to ignore them.

"Sits on furniture" (`top`): on every `commit()` the object settles onto a surface under its centre. It keeps its current shelf if that shelf is still under it, otherwise it takes the highest surface up to 1.45 m. `surfacesUnder` lists the levels and `stepLevel` backs the Shelf up / Shelf down buttons. Moving or turning a support carries its riders: `captureRiders` records them in the support's own coordinates and `carryRiders` puts them back (drag, rotate ring and the Angle field).

## Stored state

Browser localStorage, all per device:

- `capston-<flat>-work` (`capston-n1101-work`, `capston-n1102-work`): the working layout, `{v, items}`. `v` is currently 5. Loading an older version runs one-off migrations (rug sizes, first fill of shelving with books, and for N11.02 the two balcony chairs `BALC02` via `withBalcony`). Bump `v` and add a migration when a change must alter layouts people already have.
- `capston-<flat>-saved`: named layouts.
- `capston-flat`: the flat last picked.
- `capston-lang`, `capston-magnet`, `capston-screens` (glazed partitions, hidden unless `on`), `capston-nemo`.
- `capston-walked`: set after the first use of the Walk button, which pulses until then.
- `capston-hint`: set once the first-visit hint has been dismissed.
- `capston-custom`: furniture made with Claude on this device, `{key: spec}`. Working layouts and saved layouts also carry a `types` map with the spec of every custom piece they use.
- `capston-claude-key`, `capston-claude-model`: the viewer's own Anthropic API key and chosen model. The key must never be put into the page or the repository.

The layouts section first looks for `window.claude.use('db')`, which only exists when the page is hosted as a Claude artifact. On GitHub Pages that is absent and named layouts fall back to localStorage, so they are not shared between visitors. Everyone still opens on `START`.

## Claude assistant and custom furniture

A Claude tab in the dock (`#tab-ai`, `#p-ai`) lets Giray make furniture and arrange the flat from a chat, with photos. It calls the Messages API straight from the browser (`anthropic-dangerous-direct-browser-access`) with his own key, which is kept only in that browser. The tab is hidden until a key is saved; opening the page with `#claude` in the address (or `#tr-claude`) shows it so a key can be entered. Visitors without a key never see it.

- Request: `claude-opus-5-5` by default (Sonnet 5.5 selectable), adaptive thinking, effort medium, `fallbacks: "default"` with the `server-side-fallback-2026-07-01` beta on Opus (dropped automatically if the API rejects it), strict tool schemas (dropped automatically on a schema error), `tool_choice` auto. The system prompt (`AI_SYS`) and the flat's geometry (`aiGeometry`: `ROOMS`, `FIXBOX`, `SHELL.walls`, `SHELL.windows`) are cached; each user turn starts with `aiState()` (placed items and all furniture types).
- Tools (`AI_TOOLS`, run by `AI_RUN`): `place_item`, `update_item`, `remove_items`, `define_furniture`, `get_furniture_parts`, `delete_furniture_type`. Placement results report clear distances on each side (`computeGaps`) and overlaps with items, walls and fixtures (`aiOverlaps`, a separating axis test on the plan), so Claude can correct itself. All changes from one request share one Undo step.
- Room outlines: `room(en, tr, pts)` calls at the end of each shell function, in plan pixels. Sub-areas (Play corner, Kitchen) come before the open-plan room so `roomAt` names them first.
- Custom furniture is a spec: `{k, rev, n: [en, tr], c, w, d, h, y, top, parts}`, each part a box, cylinder or sphere given by its min and max on each axis in the piece's own frame at its default size, with `axis`, `tilt_x`, `rot_y`, `color`, `finish` and `texture`. `BUILD.spec` scales the parts to the item's size. Keys start with `c-`. `cleanSpec` clips anything unsafe; `registerType` adds or replaces a type unless a copy at the same or a newer revision is already known.
- Making a piece permanent: Giray presses "Copy my furniture" and pastes the JSON here. Add each spec to `SPECS` unchanged (same key and revision). Built-in specs register first; a local or layout copy with a lower or equal revision is then ignored and dropped from `capston-custom`, so every layout that uses the piece shows the built-in version. A later local edit has a higher revision and wins on that device until it is exported again.

## Walk mode and Nemo

Walk mode is a first-person camera with collision (`freeWay`), a ceiling, full-height walls and a painted backdrop outside the windows. All of those are hidden outside walk mode.

- Eye height is fixed at 1.70 m (`walk.eye`); there is no setting for it.
- The Walk button ("Walk through") is the prominent filled button at the top right of the stage, with the Display menu below it.
- While walking, `body.walking` hides the dock, the furniture sidebar, the view buttons and Undo. A Menu button opens the dock as a drawer (`body.drawer`) with Layouts only; the Furniture tab and the Walls buttons stay hidden. Leaving walk mode reopens the tab that was open before.

Nemo is the owners' dog, a 10 kg apricot labradoodle, modelled in a block style. He exists only in walk mode, is never saved, and can be toggled off.

- State machine in `think`: `wander`, `follow`, `greet`, `rest`, `toperch`, `jump`, `perched`. `decide` picks the next activity. About 11% of decisions go to his bed when one is placed (`dogBed`), which works out near 20% of his time.
- Controls in the walk panel: a "Nemo is at home" switch (`#nemo-btn`, `role="switch"`), then "Call Nemo" (or Space) brings him to the player and "Send Nemo to bed" (or B) sends him to the nearest bed. The two commands hide while he is away.
- Route finding: the floor is sampled into 15 cm cells when walk mode starts (`gridBuild`, a few hundred cells per frame), then `plan` runs A*. Obstacles are detected by casting rays upward from below the floor (`obstacleAt`). Casting downward from inside a wall finds nothing, which was a real bug.
- `moveToward` follows the route and uses short horizontal rays (`blocked`) to slide around anything the grid missed.

## Testing

There is no test suite. The method that worked: load the page in headless Chromium through Playwright with `--use-gl=swiftshader --enable-unsafe-swiftshader --ignore-gpu-blocklist`, collect `pageerror` events, and take screenshots.

- Use a copy of the page with the cdnjs URL swapped for a local `three@0.128.0` if the network is restricted.
- To reach internals, inject a line just before the `// ================= start` comment that exposes what you need on `window` (for example a camera setter, `nemo`, or `items`). Keep those hooks out of the committed file.
- Dragging by guessed pixel positions is unreliable. Drive state through injected hooks instead.
- Keyboard shortcuts are ignored while an input has focus.
- For the Claude tab, intercept `https://api.anthropic.com/v1/messages` with `page.route` and answer with scripted `tool_use` responses; the requests show exactly what would be sent.
- For Nemo, step the simulation directly with fake timestamps (`nemo.step(ts)` in a loop) and count states, then look at a still frame. Check phone width (390 px) as well as desktop.

## Known limits

- Furniture is schematic. Heights of anything not specified by the owner are assumptions.
- The magnet snaps objects to walls but is not a barrier. Objects can still overlap.
- Sitting objects use their centre point only, and do not stack on each other.
- Wall art floats above the walls in cut-away mode, because those walls stop at about 1.1 m.
- Resizing a textured rug or framed picture stretches the image.
- The window view is an invented painting, not the real outlook from the flat.
- Nemo occasionally fails to reach the player when called while he is up on furniture in a tight spot. Calling again works.
- The assistant only sees numbers, not the picture. Room outlines are approximate and wall boxes near the slanted facade are larger than the real piers.
- Custom furniture is boxes, cylinders and spheres, so it is as schematic as the rest.

## Ideas discussed but not built

- A scripted walk-through video rendered frame by frame.
- Opening on a chosen saved layout by default for first-time visitors.
- The two flats joined into one.
- More furniture and a fuller default layout for N11.02.
