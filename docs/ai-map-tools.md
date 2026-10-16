# AI Map Tools for Remere's (experimental)

This branch extends the existing Lua script menu and adds sprite-related Lua bindings. The existing Claude chat remains untouched.

## Features and files

| Feature | Menu script | What it does |
| --- | --- | --- |
| Gemini Map Builder | \`scripts/gemini_map_builder.lua\` | Call Google's official Gemini API, get validated JSON placements, read compact terrain occupancy, confirm and apply (Ctrl+Z) |
| Multi-Sprite Structure Composer | \`scripts/structure_composer.lua\` | Combine up to 220 placeable server item IDs into a reusable grid, preview sprites, save templates, stamp on map with Ctrl+Z |
| Sprite ID Inspector | \`scripts/sprite_id_inspector.lua\` | Render client appearance IDs or server item IDs; for client appearances, look up matching server item IDs and copy available item IDs |

The composer supports **Row major**, **Column major**, **Snake rows**, **Edge heuristic**, **Manual** (\`ID:x:y;ID:x:y\`), and **Saved** template modes.

The **Edge heuristic** uses a new native \`Image.autoArrange(ids, width, height)\` API. It caches all sprite edge pixels and greedily compares adjacent edges to arrange a complete grid of up to 220 items. Scores are useful hints, **not proof** of correct assembly. Only 32×32 item appearances have directly comparable edges; larger/taller sprites fall back to input order where no edge evidence exists.

Native image helpers added to \`Image\`:

- \`Image.composeItems(width, height, entries)\`: checkerboard preview of \`{id=SERVER_ITEM_ID,x=0,y=0}\` placements, anchoring sprites at the bottom-right tile.
- \`Image.edgeCost(a, b, "right"|"down")\`: heuristic edge disagreement on 32×32 appearances, or -1 if unsupported.
- \`Image.autoArrange(ids, width, height)\`: cached-edge greedy arrangement. Returns \`{entries,matchedEdges,averageCost,unsupportedSprites}\`.

A pre-existing reversed conditional in \`LuaImage::loadFromSpriteId\` was also corrected so actual multi-SQM game sprites retain their native WxH pixels and transparency. **Only the Lua image/preview layer is changed, not the regular map drawing code.**

## Test the exact range from the project

Your sample range **46967–47146** contains **180 IDs**. A possible rectangular shape is **15×12** or **12×15**, but **this does not establish the real dimensions**.

1. Open a copy of an OTBM with the matching Canary item definitions/client assets loaded.
2. Run **Scripts > Sprite ID Inspector**, choose *Server item ID* and 46967–47146. If a number is not a server item, switch to *Client appearance ID* and examine matching server items.
3. Copy the valid **server item IDs**. They are distinct from raw client sprite/appearance IDs.
4. Run **Scripts > Multi-Sprite Structure Composer**, paste the IDs, choose 15×12 and test *Row major*. Preview the generated structure. Try 12×15, *Snake rows*, or *Edge heuristic* to compare. Manual \`ID:x:y\` is supported when the layout is known.
5. To save a correct arrangement without changing the map, enter a unique template name and choose **Salvar template** in the preview dialog. The saved structure appears later with mode **Saved**.
6. To place a structure, choose **Aplicar no mapa** and confirm the map write. With collision blocking enabled, placement aborts if the footprint would collide with existing objects or replace existing ground. Newly placed non-ground objects require ground tiles to exist.
7. Review the result. Ctrl+Z is supported because the changes run in a Lua transaction. Save the OTBM only after inspecting it.

**Position transforms are not sprite rotations.** Flipping or rotating offsets does not rotate the actual artwork or choose alternate wall tiles.

## Installing and building

**The Gemini Map Builder Lua script** can be installed into the \`scripts\` folder of an existing Canary 4.0 Remere's executable that contains the same Lua APIs.

**The Structure Composer and Sprite Inspector require a new executable built from this branch** to access the new Lua image helpers and corrected renderer. Copying only the Lua scripts will not be enough to test all features.

For Windows, use a Visual Studio toolchain supporting C++23 and vcpkg. From the checked-out branch:

\`\`\`powershell
$env:VCPKG_ROOT = "C:\path\to\vcpkg"
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DCMAKE_TOOLCHAIN_FILE="$env:VCPKG_ROOT\scripts\buildsystems\vcpkg.cmake"
cmake --build build --config Release --parallel
\`\`\`

Use a separate build directory and back up the currently working editor before replacing any binaries. The repository uses CMake with Lua, sol2, wxWidgets and dependencies declared in \`vcpkg.json\`. Detailed build steps will vary with installed Windows compiler and vcpkg versions.

## Planned checks for Windows

1. The new branch compiles with \`source/lua/lua_api_image.cpp\` and \`.h\` changes.
2. The Scripts menu discovers the three Lua scripts from its executable-side scripts directory.
3. Sprite Inspector identifies server IDs and appearance IDs correctly; invalid IDs render safely.
4. Composite preview preserves tall/wide sprite transparency and bottom-right anchoring.
5. A small 4×4 row-major fixture places in the correct coordinates. Ctrl+Z restores map.
6. Test a 15×12 candidate from the 180-ID range. Verify heuristics visually, and try manual arrangement where it fails.
7. Preview-only and Save-template actions do not touch the map; Saved template can be reused later.
8. Overlap/ground preflight blocks unsafe writes, including multi-piece placement.
9. Gemini Builder prompts and JSON validation work with a real Gemini key (billing/rate limits may apply).

**No Windows build, GUI run or live API request has been performed in this chat.** This is an implementation awaiting those tests. Keep the PR as draft until it passes them.

## Limits and safety

- AI-generated Lua code is **never executed**. Structured JSON is validated and requires explicit user confirmation.
- Layout guesses cannot reliably reconstruct every multipart asset from ID order or 32×32 edge similarities. Asset metadata or exact per-piece positions are still needed for guaranteed results.
- Avoid enabling overwrite until tested in a disposable map. Default collision checking aborts instead of silently overwriting tiles.
- Gemini API is separate from the Google Antigravity editor's subscription or credits; this code does not use an unverified private API.
- No confidential keys are committed. The optional key persistence in the original Gemini Builder is plain editor storage; leave disabled on shared machines.
