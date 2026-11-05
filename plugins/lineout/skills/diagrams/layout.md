# Layout rules

Lineout draws exactly what the coordinates say, so the layout has to be computed. All values are world units; `position` is a node's top-left corner.

## Grid
- Use multiples of 8 for every position and size.
- Icon nodes are 64×64. Their label is drawn centered below the icon, wraps at 160 units wide and takes about 20 units per line.
- Put icons in a grid of columns 192 apart and rows 160 apart: column `i` at `x = x0 + 192 * i`, row `j` at `y = y0 + 160 * j`. That leaves room for a two-line label under each icon and keeps labels of neighboring icons from touching.
- Leave a column or row empty rather than squeezing nodes closer.

## Flow
- Put the main request or data flow left to right: clients and users on the left, entry points (DNS, CDN, load balancer, API gateway) next, then compute, then data stores on the right.
- Put secondary flows (async processing, monitoring, CI/CD) on a row below the main flow.
- Keep a component that talks to many others in the middle of them, so arrows stay short.

## Groups
- List group nodes before the nodes inside them, outer groups before inner groups.
- Padding inside a group, measured from the group's edge to the icons inside it:
  - left and right: 64 (labels stick out 48 on each side of an icon),
  - top: 72 (the group's icon and title take the first 28),
  - bottom: 64 below the lowest icon (room for its label).
- For a group around columns `a..b` and rows `c..d` of the grid:
  - `x = x0 + 192a - 64`, `width = 192(b - a) + 64 + 128`
  - `y = y0 + 160c - 72`, `height = 160(d - c) + 64 + 136`
- Nested groups (Region > VPC > subnet) use the same padding around the inner group's rectangle instead of an icon: 32 on the sides and bottom, 72 on top. Grow the outer group's grid cells to fit.
- Anything whose rectangle lies inside a group moves with the group in the editor, so don't let unrelated nodes overlap a group.
- Common nesting: `AWS-Cloud` > `Region` > `Virtual-private-cloud-VPC` > `Public-subnet` / `Private-subnet`. Look each one up with `search_icons`; group icons have `type: "group"`.

## Arrows
- Attach both ends to nodes: `{ "nodeId": ..., "side": ..., "offset": 0 }`. Use free `{ x, y }` endpoints only for annotations.
- Horizontal neighbors connect right → left; vertical neighbors connect bottom → top. For diagonal pairs, leave the source from the side facing the target's column (right or left) and enter the target from the side facing the source's row (top or bottom).
- When several arrows leave or enter the same side, spread them with offsets such as -16, 0 and 16 so they don't share a line.
- An arrow can attach to a group's edge to mean "the whole group".
- Point arrows the way the request or data moves. Show two-way traffic with one arrow in the direction of the call, not two arrows.
- Arrow labels are 1–3 words (`HTTPS`, `Invoke`, `SQL`, `Events`), placed at `"position": 0.5, "side": "over"`. Label only the arrows where the label adds information.
- An arrow that leaves or enters an icon's bottom side runs under the icon's label, because labels are drawn on a solid box. For labeled icons, prefer the other sides unless the bottom gives the clearest route.

## Arrow routes and frozen segments
Without `frozen`, Lineout routes each arrow automatically: it leaves the start side, bends once halfway between its two nodes and enters the end side. The auto-route avoids only the arrow's own two nodes. It runs straight through any other node, label or group border in the way. `frozen` pins the route instead.

### When to add `frozen`
Picture the auto-route as running inside the rectangle spanned by the two endpoints. Give the arrow a frozen route when:
- another node, its label or a group border it doesn't belong to lies in that rectangle, which usually happens between nodes that aren't neighbors,
- the bend halfway would fall on a node or label instead of a gap,
- several arrows would end up on top of each other, or you want them to share one neat corridor,
- the arrow has to go around something, such as a U-turn between two nodes in the same row, or a return path from right to left.

Leave `frozen` out between neighbors whose rectangle is empty; the auto-route is already clean there.

### How a frozen route is built
- Each entry is a whole line, not a segment: `{ "orientation": "vertical", "value": x }` is the line `x = value`, and `{ "orientation": "horizontal", "value": y }` is the line `y = value`.
- The route leaves the start side, then follows each line in list order, turning where consecutive lines cross, then enters the end side. Leaving or entering a left or right side is a horizontal move; a top or bottom side is a vertical move.
- Alternate orientations so that every turn is a real corner, counting the start and end moves. Leaving a right side (horizontal), the first line is vertical; entering a left side (horizontal), the last line is vertical too. So right → left takes 1 or 3 lines (`V` or `V H V`), and right → top takes 2 (`V H`).
- If two consecutive lines share an orientation (including a first line that is parallel to the start move), the router joins them with a jog it places itself, only 16 units from the node. That looks cramped, so don't rely on it.
- Don't put a line behind the node the arrow is leaving (left of a node when leaving its right side); the route then detours around the node. A line that the attached endpoint already sits on is ignored.
- One to three lines is almost always enough.

### Where to put the lines
Run lines through the empty corridors of the grid from "Grid" above, never through icons or labels:
- Vertical corridor between columns `i` and `i + 1`: `x = x0 + 192i + 128`.
- Horizontal corridor between rows `j` and `j + 1`: `y = y0 + 160j + 136`.
- Above the first row: `y = y0 - 40`. Left of the first column: `x = x0 - 80`. Stay at least 24 away from group borders and run on the side of the border where both endpoints are.
- When several arrows share a corridor, move each line 16 apart (`x - 16`, `x`, `x + 16`), in the same order as their endpoints, so the lines don't cross each other.

### Example
In [examples/serverless-api.json](examples/serverless-api.json) (`x0 = 192`, `y0 = 120`), an arrow from `cdn` (column 0, row 0) to `fn` (column 2, row 1) would auto-route through `site` and `api`. Route it around them, down the corridor after column 0, along the corridor between the rows, then down the corridor after column 1:

```json
{
  "id": "a6",
  "start": { "nodeId": "cdn", "side": "right", "offset": 0 },
  "end": { "nodeId": "fn", "side": "left", "offset": 0 },
  "frozen": [
    { "orientation": "vertical", "value": 320 },
    { "orientation": "horizontal", "value": 256 },
    { "orientation": "vertical", "value": 512 }
  ]
}
```

The route is (256,152) → (320,152) → (320,256) → (512,256) → (512,312) → (576,312), with no crossings. A U-turn between `api` and `table` in the same row: leave `api`'s bottom, then `[{ "orientation": "horizontal", "value": 400 }]`, then enter `table`'s bottom.

### Keeping frozen routes valid
- Frozen lines stay where they are when nodes move. When `edit_page` moves a node, recompute the frozen lines of every arrow attached to it, or remove them if the new auto-route is clean.
- Keep frozen lists unchanged on arrows you didn't touch; users created them by dragging segments in the editor.

## Text nodes
- Use `"type": "text"` nodes (160×32 by default) for titles and short notes, placed outside groups, usually above the diagram at `y0 - 96`.

## Checking for overlaps
Two nodes overlap when `a.x < b.x + b.width && b.x < a.x + a.width && a.y < b.y + b.height && b.y < a.y + a.height`, except when one is a group that contains the other. When you add a node to an existing page, find a free cell with this test. If there's none, move whole columns or rows to the right or down with `edit_page`, together with the groups around them.
