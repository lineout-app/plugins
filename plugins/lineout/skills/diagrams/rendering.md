# Rendering images with the `lineout` CLI

The `lineout` command renders diagram JSON to SVG, PNG or JPG locally. It uses the same arrow router and drawing code as the editor, so the image matches what users see in Lineout. It needs no account and no MCP server.

## Check that it's available
Run `lineout --version`. If the command is missing, ask the user before installing it. It's published on PyPI as `lineout-cli`:

```bash
uv tool install "lineout-cli[jpg]"
```

Without uv, use `pipx install "lineout-cli[jpg]"`. Never install the package named `lineout` (`pip install lineout`, `uvx lineout`): that name on PyPI belongs to an unrelated package. If the user declines or the install fails, tell them and skip rendering.

This skill needs `lineout` 0.2.0 or newer. If `lineout --version` prints an older version, ask the user to upgrade it with `uv tool upgrade lineout-cli` (or `pipx upgrade lineout-cli`) before rendering. When the CLI prints that a newer `lineout-cli` is available, mention it to the user once.

## Input: one page per file
The CLI renders a single page: `{ "nodes": [...], "arrows": [...] }`, the same format the MCP tools use. A `{ "pages": [...] }` document fails with `nodes must be an array`. Write one file per page. To render a page you read with `get_diagram`, take that page's `nodes` and `arrows` and leave out its `index` and `name`.

## Commands
```bash
lineout page.json -o page.png                 # format from the suffix: .svg, .png, .jpg
lineout page.json                             # writes page.svg next to the input
lineout page.json -o page.png --theme dark    # light (default) or dark
lineout page.json -o page.png --transparent   # no background, svg and png only
lineout page.json -o page.png --scale 1       # pixel ratio for png and jpg, default 2
```

Errors name the field at fault, like the MCP tools do (`lineout: error: arrows[2].end.nodeId "db" does not match any node`). The command exits with status 1.

## Checking a layout by eye
After writing or changing a page, render it to PNG in a temporary directory and look at the image (read the file). The coordinate checks in [layout.md](layout.md) miss things that are obvious in the picture. Look for:
- arrows crossing icons, labels or group borders they don't belong to,
- arrow labels sitting on a group border or on another line (move them with `position`, for example 0.25 or 0.75),
- arrows running under an icon's label (attach them to another side, or route them with `frozen`),
- nodes or labels touching, groups that clip their children, and big empty areas.

Fix what you find with `edit_page` (or in the JSON file), render again and repeat until the picture is clean. Delete the temporary files when you're done.

## Producing image files for the user
When the user wants an image, for example an architecture diagram for a README or documentation:
- Save the page JSON next to the image (`docs/architecture.json` and `docs/architecture.svg`), so the image can be regenerated and the JSON can be imported into Lineout by dropping it on a project.
- Prefer SVG for documentation (sharp at any size, small, text stays text) and PNG where SVG isn't supported. Use `--theme dark` only when asked.
- If the diagram also lives in Lineout, say which diagram and page the file was rendered from, since the file won't follow later edits.
