# pagesnap-to-figma

A Claude Code skill that lays out screenshots from a local folder into a Figma section as a documented user flow.

## What it does

Give it a folder of screenshots and a Figma section URL — it will:

- Sort screenshots chronologically
- Create one frame per screenshot (1024px tall, proportional width)
- Arrange frames left-to-right with 80px gaps
- Add a title (the folder name) above each row
- Upload the actual images into the frames
- Extend the section automatically if needed
- Take a screenshot to confirm the result

## Requirements

- [Claude Code](https://claude.ai/code)
- Figma MCP enabled in Claude Code

## Installation

Copy the skill folder into your personal Claude skills directory:

```bash
cp -r pagesnap-to-figma ~/.claude/skills/pagesnap-to-figma
```

The skill is available immediately — no restart needed.

## Usage

Trigger it by telling Claude something like:

- "Put my screenshots in Figma"
- "Lay out screenshots from ~/Desktop/MyFlow into this Figma section: [URL]"
- "Document this flow in Figma from my screenshots folder"
- "Add my screenshots to Figma"

Or just paste a folder path and a Figma section URL — Claude will recognize the intent automatically.

## Layout

```
[existing content — preserved, never overwritten]
  ↓ 1600px gap (if section already has content)

  [Title — folder name, Arimo Bold 60px]
  ↓ 100px

  [Frame 1] — 80px — [Frame 2] — 80px — [Frame 3] …
  (height: 1024px, width: proportional to original)

1000px padding on left, top, and bottom.
Section resizes automatically to fit new content.
```
