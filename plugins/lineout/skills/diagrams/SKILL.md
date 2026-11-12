---
name: diagrams
description: Create, update and review cloud architecture diagrams in Lineout through its MCP tools. Use when the user asks to draw, diagram, visualize or document an architecture or system design in Lineout, to turn infrastructure code (Terraform, CDK, CloudFormation, SAM, Serverless, Kubernetes or docker-compose files) or a written description into a Lineout diagram, to change or extend an existing Lineout diagram, to explain what a Lineout diagram shows, or to render a Lineout diagram or diagram JSON file to an SVG, PNG or JPG image (for example for a README or documentation).
allowed-tools: mcp__plugin_lineout_lineout__list_workspace mcp__plugin_lineout_lineout__list_team_members mcp__plugin_lineout_lineout__get_diagram mcp__plugin_lineout_lineout__get_share_link mcp__plugin_lineout_lineout__get_format_guide mcp__plugin_lineout_lineout__search_icons Bash(lineout *)
---

# Lineout diagrams

Lineout is a collaborative editor for AWS architecture diagrams, organized as team → project → diagram → pages. Two tools are involved:

- **The `lineout` MCP server** reads and writes diagrams in Lineout. If its tools are missing or return an authorization error, tell the user to run `/mcp` and authenticate the `lineout` server. If a tool result has a `notice`, for example that this plugin is out of date, pass it on to the user once.
- **The `lineout` command line tool** renders diagram JSON to SVG, PNG or JPG locally, with the same drawing code as the editor. Use it to check layouts by eye and to produce image files. See [rendering.md](rendering.md).

Without the MCP server you can still produce diagram JSON files and images with the CLI, following steps 2–4 and "Image files" below. For anything stored in Lineout you need the server.

## Workflow

### 1. Find where the diagram lives
- Call `list_workspace`. It returns team, project and diagram ids, page names and editor URLs.
- When the user names a diagram, match it by name. If several match, or none, ask instead of guessing.
- For a new diagram, use the project the user named. If they named none, propose one (or the personal team's "Examples" project) and confirm before creating anything.

### 2. Understand the content before drawing
- **From a description:** list the components, the boundaries that group them (account, region, VPC, subnet, cluster) and the flows between them before writing any JSON.
- **From code in the repository:** read the infrastructure files (`*.tf`, CDK stacks, `template.yaml`, `serverless.yml`, Kubernetes manifests, `docker-compose.yml`) and extract resources, containment and connections (event sources, integrations, security group rules, environment variables that reference other resources). Leave out plumbing such as IAM roles, policies, log groups and parameters unless the user asks for them. Mention what you left out.
- **For an existing diagram:** call `get_diagram` with the page index and read its nodes and arrows. Never edit a page you have not read in this conversation.

### 3. Pick icons
- Call `get_format_guide` once per conversation before writing content.
- Call `search_icons` for every service. Use the returned `icon` value exactly. Never invent icon names.
- Always prefer service icons (sections without "resources" in the name) over resource icons when available.
- Results with `type: "group"` are only for group nodes: AWS Cloud, Region, VPC, Public subnet, Private subnet, Auto Scaling group and similar.

### 4. Lay it out
Follow [layout.md](layout.md). In short: one main flow left to right, an 8-unit grid, 192 units between columns and 160 between rows, groups padded 64 units (72 at the top), and arrows attached to the sides that face each other. [examples/serverless-api.json](examples/serverless-api.json) is a complete, valid page to copy from.

Arrows route themselves, but the auto-route only avoids the arrow's own two nodes. For every arrow between nodes that aren't neighbors, check whether its path would cross another node, label or group border. If it would, pin it with `frozen` lines through the empty corridors between columns and rows, as described in "Arrow routes and frozen segments" in [layout.md](layout.md).

For things that aren't AWS services, such as process steps, decisions, actors or notes, use shape nodes (`"type": "shape"` with a `shape` like `"rectangle"`, `"diamond"` or `"cylinder"`) with the text as their label, as described in `get_format_guide`. For notes and annotations, use a `"sticky-note"` shape with a fill such as `"yellow"` and a `"transparent"` border color. Size each shape for its text.

Leave `style` off nodes and arrows unless the user asks for colors, fills, fonts, text alignment, line styles or arrowheads, or the page you're editing already uses them. The fields are in "Styles" in `get_format_guide`. Prefer palette color names over hex values, and keep `style` on items you upsert with `edit_page`, because an upsert replaces the whole item.

### 5. Write
- **New diagram:** call `create_diagram` with the whole document in one call: `{ "pages": [{ "name": ..., "nodes": [...], "arrows": [...] }] }`.
- **Additional view:** call `add_page` with the new page's content (for example, a network view next to a request-flow view).
- **Change an existing page:** call `edit_page` with only the nodes and arrows that change, sent as complete objects. Keep existing ids. Pass `remove` with ids to delete items; arrows attached to a removed node go with it. To make room, move existing nodes by upserting them with new positions.
- **`replace_page`:** use only to redraw a page from scratch, and only after the user agrees. Keep the ids of items that survive, so collaborators' views don't jump.
- If a call fails, read the error path (for example `node "fn".position.x must be a number`), fix that item and retry. Don't drop content to get past a validation error.

### 6. Check and report
- Call `get_diagram` for the page you wrote. Check that every planned node and arrow is there and that no two nodes overlap (compare positions and sizes).
- Check every arrow's route: a frozen route through its listed lines, otherwise the rectangle between its endpoints. If a route crosses a node, a label or a group border it shouldn't, fix it with `edit_page` by adding or adjusting `frozen` lines.
- If the `lineout` CLI is installed, render the page to PNG and look at the image, as described in "Checking a layout by eye" in [rendering.md](rendering.md). Fix what looks wrong and render again.
- Reply with the diagram's URL and a short summary of what you drew or changed, plus any assumptions you made.

## Image files
When the user wants an image or a diagram file in the repository rather than (or as well as) a diagram in Lineout:
- Build the page as in steps 2–4 and write it as a single-page JSON file `{ "nodes": [...], "arrows": [...] }`. For a diagram already in Lineout, take the page from `get_diagram`.
- Render it with `lineout page.json -o page.svg` (or `.png`, `.jpg`), check it by eye and keep the JSON next to the image. Details are in [rendering.md](rendering.md).
- Treat the CLI warning `icon not found` as an error: the icon renders as a blank space. Without the MCP server's `search_icons`, take icon names from the files bundled with the CLI. They live under `_assets/aws` in the installed `lineout` package, which runs in its own Python environment, so use the CLI's own interpreter to find the folder: `"$(head -1 "$(command -v lineout)" | cut -c3-)" -c "import lineout, pathlib; print(pathlib.Path(lineout.__file__).parent / '_assets/aws')"`. Then search it with `find <folder> -iname '*lambda*'`. An icon's name is its file name without the `Arch_`/`Res_` prefix, the size suffix (`_64`, `_48`, `_32`) and `.svg`, so `Arch_AWS-Lambda_64.svg` is `AWS-Lambda`. Skip files ending in `_Dark`; they're the dark-theme variants of the same icon.

## Safety
- Page edits appear live for everyone who has the diagram open, and nobody can undo them from the editor. Prefer small `edit_page` calls over rewrites.
- Ask before `delete_diagram`, `delete_project`, `delete_page`, `revoke_share_link` and `replace_page` on a page that already has content, and say what will be lost.
- `share_diagram` creates a link that lets anyone who signs in view or edit the diagram. Only create or change one when the user asks.
- If `get_diagram` reports `role: "viewer"`, you can read the diagram but not change it. Say so instead of attempting writes.
