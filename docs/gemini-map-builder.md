# Gemini AI Map Builder (experimental)

This feature adds `scripts/gemini_map_builder.lua` as a separate item in the **Scripts** menu. The existing `claude_agent.lua` is unchanged.

## What is implemented

- Talks to the **official Google Gemini API** over HTTPS using the editor's existing Lua HTTP streaming API. **It is not an undocumented Antigravity API**, and Antigravity app quotas do not provide Gemini API credentials.
- Requests a structured JSON tile plan (relative X/Y + server item IDs). It **never executes model-generated Lua**.
- Restricts AI output to item IDs explicitly allowed by the user and validated through `Items.exists` and `Items.getInfo`.
- Includes a compact region grid in the prompt (`.` empty, `g` ground-only, `X` with objects).
- Enforces map bounds, tile count, per-tile item count, unique positions, valid ground, and maximum region/ID limits.
- Default preservation mode skips existing decorated tiles and prevents ground replacement; it can add objects to tiles that already contain only ground.
- Shows a textual plan summary before application; changes run inside `app.transaction` and support Ctrl+Z.

## How to try it on Windows

1. Download this branch or copy `scripts/gemini_map_builder.lua` into the **scripts folder next to the running Remere's executable**. You can use the Scripts menu's "Open Scripts Folder" and "Reload Scripts" features, if available in your build.
2. Open a map. Use `Scripts > Gemini AI Map Builder`.
3. Get a Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey). Paste it into the script dialog. The *Save key* checkbox is **off by default**.
4. Provide one small test area, such as 8×8, and a set of **valid SERVER ITEM IDs** including a ground ID for empty tiles. You can separate IDs by commas or pass an inclusive range like `100-120`. Do not use placeholder IDs without checking your client's item database.
5. Describe what to place. Review the predicted tile count and sample placement coordinates, then click **Yes** to apply. Save your map manually after checking the result; use Ctrl+Z to undo.
6. Start on a **backup/copy** of your OTBM, not the only original file.

No C++ changes are needed for this initial script **if your executable already includes the same Lua APIs as this fork**. Pushing a Lua source file to GitHub does not update an existing Windows executable automatically.

## Important limitations

- **This version cannot infer the spatial layout of multi-part sprites from numeric IDs alone.** IDs that are consecutive are not guaranteed to be in visual grid order. Accurate multi-part assembly requires an explicit mapping (ID → relative coordinates) or sprite-image/asset analysis.
- RME places **server item IDs**, not raw client sprite IDs. A number from the client sprite atlas may not identify a placeable item. Verify the ID category first.
- The initial preview is **text only**, not a rendered image overlay.
- Up to **220 allowed IDs, 200 planned tiles, 8 items per tile**, and maximum 40×40 target region.
- The HTTP streaming worker has a ~30-second request timeout in this fork. Long generations may fail; try small regions and concise prompts.
- If you opt in to saving the key, this script uses the RME Lua storage API, **not encrypted secret storage**. Leave it unchecked on shared computers.
- The API can be rate-limited or billed separately from subscriptions to Antigravity or Gemini.
- No API call, local GUI test or Windows build has been run for this feature yet.

## Suggested manual smoke tests

| Case | Expected behavior |
| --- | --- |
| No map open | Shows an error and does not call the API |
| Invalid ID/range | Rejects before sending a request |
| Invalid/nonexistent ground on empty tiles | Rejects the layout before any edits |
| Generated out-of-bounds/duplicate tiles | Rejects without editing |
| Existing decoration + preservation checked | Skips affected tile |
| Ground-only tile + objects with preservation checked | Allows object placement |
| Cancel confirmation | No map changes |
| Accept a valid small layout | One undoable transaction; Ctrl+Z reverses it |
| Invalid/missing Gemini key or model | Returns a useful API/network error |

## Next step

Integrate the [MCP editor tools proposal](https://github.com/opentibiabr/remeres-map-editor/pull/209) or build a sprite-mapping tool so the model can inspect **real rendered sprite parts**, derive width/height and position each item correctly. Keep human confirmation before writing to the map.
