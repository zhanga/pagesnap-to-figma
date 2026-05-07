---
name: pagesnap-to-figma
description: Lays out screenshots from a local folder into a Figma section as a documented user flow. Use this skill whenever the user wants to upload screenshots to Figma, document a flow or journey in Figma from captured screenshots, place images from a folder into a Figma design file, or lay out screenshots chronologically in Figma. Triggers on phrases like "put screenshots in Figma", "lay out screenshots", "document flow in Figma", "upload screenshots to Figma section", "add my screenshots to Figma", or "create a flow in Figma from screenshots". Use it even if the user only mentions a folder name and a Figma link without explicitly asking for a "skill".
---

# PageSnap → Figma Flow Layout

Lay out screenshots from a local folder into a Figma section as a user flow: one frame per screenshot, left-to-right in chronological order, 1024 px tall (width proportional), 80 px gaps between frames, with a title above. Each row of screenshots has 1000px breathing room from the section edges (left, top, and bottom).

## Step 1 — Gather inputs

You need two things before starting:

**1a. Screenshots folder**
The user may give a folder path directly (e.g. `/Users/amy/Desktop/Classes Tool/Test`) or just a folder name (e.g. `"Test"`). If just a name, search common locations:
- `~/Desktop/**/<name>`
- `~/Downloads/**/<name>`
- `~/Documents/**/<name>`

If you find **more than one** folder with that name, list them all and ask the user which one to use — don't silently pick the first match.

If you can't locate the folder at all, ask the user where it is.

**1b. Figma section**
The user may paste a Figma URL with a `node-id` param — that's the target section. If they haven't provided one, ask:

> "Which Figma section should I place the screenshots in? Please paste the section's URL (or its node ID and file key)."

Parse the URL:
- `fileKey` → the segment after `/design/` or `/file/` (both formats exist)
- `nodeId` → the `node-id` query param, replacing `-` with `:`

**How to get the link:** Right-click the section in Figma → **Copy link**, or use the Share button. Works the same from the browser or the Mac app. Avoid copying from the address bar — the Figma Mac app may produce a `figma://` deep link that won't parse correctly.

## Step 2 — Read and sort screenshots

```bash
ls -lt "<folder>/"
```

List all `.png` and `.jpg` files. The `-t` flag orders newest first — **reverse this** to get chronological (oldest first = left-most frame).

Get each image's pixel dimensions:
```bash
sips -g pixelWidth -g pixelHeight "<file>"
```

## Step 3 — Calculate frame sizes

Target height: **1024 px**. Width scales proportionally:

```
frame_width = round(original_width * 1024 / original_height)
```

## Step 4 — Inspect the target section

Call `get_metadata` with the section's `nodeId` and `fileKey`. Note the section's canvas `x`, `y`, `width`, and `height` — you'll need these for absolute positioning.

## Step 5 — Create frames and title in Figma

**NEVER delete or overwrite existing content.** Always append below what's already there.

Use `use_figma` to:

1. Find the section across pages (iterate with `setCurrentPageAsync`).
2. **Find where existing content ends** — scan all current children and compute `maxBottom`:
   ```js
   const maxBottom = section.children.reduce((m, c) => Math.max(m, c.y + c.height), 0);
   const startY = maxBottom > 0 ? maxBottom + 1600 : 1000;
   ```
   If the section is empty, start at y = 1000 (1000px top padding). Otherwise, start 1600px below the lowest existing child.
3. Read `section.fills` to determine background luminance:
   ```js
   const fill = section.fills?.[0];
   const { r, g, b } = fill?.color ?? { r: 1, g: 1, b: 1 };
   const luminance = 0.2126 * r + 0.7152 * g + 0.0722 * b;
   const textColor = luminance < 0.5
     ? { r: 1, g: 1, b: 1 }  // white on dark
     : { r: 0, g: 0, b: 0 }; // black on light
   ```
4. Create a **Text node** for the title:
   - Characters = folder name (e.g. `"Test"`)
   - Font: **Arimo Bold, 60px** (Arial is not available in Figma's cloud library; Arimo is metrically identical)
   - Color: `textColor` from luminance check above
   - Position: `title.x = 1000`, `title.y = startY`
   - Do NOT add `section.x` or `section.y` — children use section-local coordinates
5. Create one **Frame** per screenshot:
   - Width = calculated proportional width, Height = 1024
   - Position: `frame.y = startY + 60 + 40`, `frame.x` starts at 1000 then increments by `prev_width + 80`
   - Placeholder fill: light gray `{r:0.92, g:0.92, b:0.94}`
   - Corner radius: 8
   - Append each frame to the section
6. **Extend the section if needed** — after appending all frames, check if content exceeds the section bounds and resize:
   ```js
   const newBottom = framesY + 1024 + 1000;  // frames bottom + 1000px padding
   const newRight = 1000 + frames.reduce((sum, f) => sum + f.w + 80, 0);
   const newH = Math.max(section.height, newBottom);
   const newW = Math.max(section.width, newRight);
   if (newH !== section.height || newW !== section.width) {
     section.resizeWithoutConstraints(newW, newH);
   }
   ```
7. Return all created frame IDs and the title ID.

Key rules for the `use_figma` call:
- Use `await figma.setCurrentPageAsync(page)` to switch pages (sync setter throws)
- Set frame fills as a new array, never mutate in place
- Colors are 0–1 range
- `return` all created node IDs
- **Never use `section.x` / `section.y` as position offsets for children** — those values are in the parent section's coordinate space, not the child's

## Step 6 — Upload images

For each screenshot (in order), call `upload_assets` with:
- `fileKey` = the Figma file key
- `nodeId` = the corresponding frame ID from Step 5
- `count` = 1
- `scaleMode` = `"FILL"`

This returns a `submitUrl`. **POST the raw image bytes** to it:

```bash
curl -s -X POST \
  -F "file=@<path-to-image>;type=image/png" \
  "<submitUrl>"
```

You can run all uploads in parallel (one `upload_assets` + one `curl` per image, all at once).

## Step 7 — Verify

Take a screenshot of the section using `use_figma`:

```js
const section = /* find by id */;
await section.screenshot({ scale: 0.15 });
return { frameCount: <n>, sectionId: section.id };
```

Show the screenshot to the user and confirm everything looks right.

## Layout summary

All coordinates are **relative to the section's top-left corner (0, 0)**:

```
[existing content — preserved, never touched]
  ↓ 1600px gap
  [Title text — folder name, Arimo Bold 60px, color by background]
  ↓ 60px (title height) + 40px gap
  [Frame 1] —80px— [Frame 2] —80px— [Frame 3] …
  (all frames: height 1024px, width proportional to original)

If section is empty, title starts at y=1000 (1000px top padding).
1000px left padding for all rows. 1000px bottom padding after last frame row.
Section width/height extended automatically if new content won't fit.
```

## Error handling

- **Folder not found**: ask the user for the path.
- **Multiple folders with the same name**: list the full paths and ask which one to use.
- **No images in folder**: tell the user and stop.
- **Figma section not found**: re-check the node ID and page; if still missing, ask the user to confirm the URL.
- **Upload fails**: report the specific image and curl response; offer to retry.
- **Section too small**: automatically extend with `resizeWithoutConstraints` — never warn or stop for this.
