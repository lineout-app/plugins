---
name: diagrams
description: Create, update and review cloud architecture diagrams and UML diagrams in Lineout through its MCP tools. Use when the user asks to draw, diagram, visualize or document an architecture or system design in Lineout, to draw a UML sequence, activity, use case or state diagram in Lineout, to turn infrastructure code (Terraform, CDK, CloudFormation, SAM, Serverless, Kubernetes or docker-compose files) or a written description into a Lineout diagram, to change or extend an existing Lineout diagram, to explain what a Lineout diagram shows, or to render a Lineout diagram or diagram JSON file to an SVG, PNG or JPG image (for example for a README or documentation).
allowed-tools: mcp__plugin_lineout_lineout__list_workspace mcp__plugin_lineout_lineout__list_team_members mcp__plugin_lineout_lineout__get_diagram mcp__plugin_lineout_lineout__get_share_link mcp__plugin_lineout_lineout__get_format_guide mcp__plugin_lineout_lineout__search_icons mcp__plugin_lineout_lineout__get_cost_estimate mcp__plugin_lineout_lineout__get_pricing_inputs Bash(lineout *)
---

# Lineout diagrams

Lineout is an AWS-native, collaborative editor for cloud architecture diagrams, organized as team → project → diagram → pages. Two tools are involved:

- **The `lineout` MCP server** reads and writes diagrams in Lineout. If its tools are missing or return an authorization error, tell the user to run `/mcp` and authenticate the `lineout` server. If a tool result has a `notice`, for example that this plugin is out of date, pass it on to the user once.
- **The `lineout` command line tool** renders diagram JSON to SVG, PNG or JPG locally, with the same drawing code as the editor. Use it to check layouts by eye and to produce image files. See [rendering.md](rendering.md).

Without the MCP server you can still produce diagram JSON files and images with the CLI, following steps 2–4 and "Image files" below. For anything stored in Lineout you need the server.

## Workflow

### 1. Find where the diagram lives
- Call `list_workspace`. It returns team, project and diagram ids, page names and editor URLs, your `role` in each team (`owner`, `admin` or `member`) and each team's `plan`, plus `locked` diagrams and `viewOnly` teams (see "Plan limits" below).
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
- Use each cloud's own icons for its services: AWS icons for AWS, Google Cloud icons (names starting with `GCP_`) for Google Cloud, Azure icons (names starting with `Azure_`) for Azure. Put the cloud's name in every search, `aws`, `gcp` or `azure`, such as `aws load balancer`, `gcp bigquery` or `azure cosmos db`: the search ranks the closest matches from every cloud together, so without it another cloud's icon can come first. Google Cloud has its own icon for only some products; for the rest, use the category icon that the search returns for the product's name, such as Serverless Computing for `gcp cloud functions`.
- For things that aren't cloud services, such as on-premises servers and network gear, user devices, people, places and third-party systems, search with the word "generic" in the query, such as `generic firewall` or `generic laptop`, and use the generic icons (names starting with `Generic_`).
- Results with `type: "group"` are only for group nodes: AWS Cloud, Region, VPC, Public subnet, Private subnet, Auto Scaling group and similar.

### 4. Lay it out
For a new page that is mostly a flow, you can write a Mermaid flowchart instead of JSON and let Lineout lay it out: pass its text as `document` to `create_diagram` or as `content` to `add_page`, as described in "Mermaid diagrams" in `get_format_guide`. Give icon nodes the names `search_icons` returns, as in `fn@{ icon: "AWS-Lambda", label: "Orders" }`, and groups theirs with a line such as `vpc@{ icon: "Virtual-private-cloud-VPC" }`. The Mermaid ids become the node ids, so you can price or adjust the nodes afterwards. Write JSON instead when the page needs exact positions, sticky notes or styles beyond simple colors, and always to change an existing page: `edit_page` and `replace_page` take JSON only. Check a page made from Mermaid as in step 6, like any other.

Otherwise, follow [layout.md](layout.md). In short: one main flow left to right, an 8-unit grid, 192 units between columns and 160 between rows, groups padded 64 units (72 at the top), and arrows attached to the sides that face each other. [examples/serverless-api.json](examples/serverless-api.json) is a complete, valid page to copy from.

Arrows route themselves, but the auto-route only avoids the arrow's own two nodes. For every arrow between nodes that aren't neighbors, check whether its path would cross another node, label or group border. If it would, pin it with `frozen` lines through the empty corridors between columns and rows, as described in "Arrow routes and frozen segments" in [layout.md](layout.md).

For things that aren't AWS services, such as process steps, decisions, actors or notes, use shape nodes (`"type": "shape"` with a `shape` like `"rectangle"`, `"diamond"` or `"cylinder"`) with the text as their label, as described in `get_format_guide`. For notes and annotations, use a `"sticky-note"` shape with a fill such as `"yellow"` and a `"transparent"` border color. Size each shape for its text.

For a new UML diagram, write it in Mermaid and let Lineout lay it out, as described in "Mermaid diagrams" in `get_format_guide`: a `sequenceDiagram`, a `stateDiagram-v2`, a `usecase-beta` use case diagram, or for an activity diagram a flowchart with the `start`, `stop`, `fork` and `cross-circ` shapes. To change an existing UML page, or for details Mermaid can't express, write JSON following "UML diagrams" in `get_format_guide`: lifelines with messages attached at matching offsets, actors, filled circles, bullseyes and bars, frames (groups with `"frame"` and no icon) for fragments, system boundaries and composite states, and `"headShape"` for open and hollow arrowheads. UML diagrams need no icons, so skip `search_icons` unless the user wants cloud services in them.

Leave `style` off nodes and arrows unless the user asks for colors, fills, fonts, text alignment, line styles or arrowheads, or the page you're editing already uses them. The fields are in "Styles" in `get_format_guide`. Prefer palette color names over hex values, and keep `style` and `cost` on items you upsert with `edit_page`, because an upsert replaces the whole item.

### 5. Write
- **New diagram:** call `create_diagram` with the whole document in one call: `{ "pages": [{ "name": ..., "nodes": [...], "arrows": [...] }] }`, or the text of a Mermaid flowchart, sequence, state or use case diagram for one page that Lineout lays out.
- **Additional view:** call `add_page` with the new page's content, as JSON or Mermaid (for example, a network view next to a request-flow view).
- **Change an existing page:** call `edit_page` with only the nodes and arrows that change, sent as complete objects. Keep existing ids. Pass `remove` with ids to delete items; arrows attached to a removed node go with it. To make room, move existing nodes by upserting them with new positions. A page with a `source` in `get_diagram` is synced from a repository, and one with `locked: true` is locked; see "Synced pages" and "Locked pages" below.
- **`replace_page`:** use only to redraw a page from scratch, and only after the user agrees. Keep the ids of items that survive, so collaborators' views don't jump.
- If a call fails, read the error path (for example `node "fn".position.x must be a number`, or `line 4: no icon is called lambda` for Mermaid), fix that item and retry. Don't drop content to get past a validation error. A plan-limit or read-only error is different: don't retry it, follow "Plan limits" below.

### 6. Check and report
- Call `get_diagram` for the page you wrote. Check that every planned node and arrow is there and that no two nodes overlap (compare positions and sizes).
- Check every arrow's route: a frozen route through its listed lines, otherwise the rectangle between its endpoints. If a route crosses a node, a label or a group border it shouldn't, fix it with `edit_page` by adding or adjusting `frozen` lines.
- If the `lineout` CLI is installed, render the page to PNG and look at the image, as described in "Checking a layout by eye" in [rendering.md](rendering.md). Fix what looks wrong and render again.
- Reply with the diagram's URL and a short summary of what you drew or changed, plus any assumptions you made.

## Image files
When the user wants an image or a diagram file in the repository rather than (or as well as) a diagram in Lineout:
- Build the page as in steps 2–4 and write it as a single-page JSON file `{ "nodes": [...], "arrows": [...] }`. For a diagram already in Lineout, take the page from `get_diagram`.
- Render it with `lineout page.json -o page.svg` (or `.png`, `.jpg`), check it by eye and keep the JSON next to the image. Details are in [rendering.md](rendering.md).
- Treat the CLI warning `icon not found` as an error: the icon renders as a blank space. Without the MCP server's `search_icons`, take icon names from the files bundled with the CLI. They live under `_assets/aws` in the installed `lineout` package, which runs in its own Python environment, so use the CLI's own interpreter to find the folder: `"$(head -1 "$(command -v lineout)" | cut -c3-)" -c "import lineout, pathlib; print(pathlib.Path(lineout.__file__).parent / '_assets/aws')"`. Then search it with `find <folder> -iname '*lambda*'`. An icon's name is its file name without the `Arch_`/`Res_` prefix, the size suffix (`_64`, `_48`, `_32`) and `.svg`, so `Arch_AWS-Lambda_64.svg` is `AWS-Lambda`. Skip files ending in `_Dark`; they're the dark-theme variants of the same icon. The Google Cloud, Azure and generic icons are in `_assets/gcp`, `_assets/azure` and `_assets/generic` next to it, named the same way without a size: `GCP_BigQuery.svg` is `GCP_BigQuery`, `Azure_Virtual-Machine.svg` is `Azure_Virtual-Machine` and `Generic_Router_Light.svg` is `Generic_Router`.

## Cost estimates
Lineout estimates the monthly cost of the AWS service icons inside an AWS Cloud group, with AWS Pricing Calculator data. Use it when the user asks what an architecture costs or wants to change the assumptions behind the estimate.
- **Read the estimate:** call `get_cost_estimate` for the page. It returns the total in USD, one row per service with its cost per copy and for all its copies, and issues such as services outside the AWS Cloud group, which aren't counted. An EC2 row with `regionFee` also carries AWS's per region fee for Dedicated Instances, which is charged once per region, so mention it when that row looks high. Quote the figures as estimates.
- **Change the assumptions:** call `get_pricing_inputs` for the node first. It lists the region of an AWS Cloud or Region group, or a service's templates and the inputs the pricing panel shows, with their `id`, `kind`, current value, `units`, `options` and `limits`. Then call `set_pricing` with those ids: `{ "values": { "<id>": { "value": "5000000", "unit": "perMonth" } } }`. Use option and unit ids, not labels. `null` resets an input to its default.
- Set the region on the AWS Cloud group, or on a Region group inside it, never on a service. Use `count` for identical copies of a service, such as 20 Fargate tasks, and `excluded: true` to leave a service out.
- Inputs depend on each other: changing one can show or hide others. After a change, call `get_pricing_inputs` again before setting inputs that weren't listed before.
- `set_pricing` changes nothing when any value is invalid, and lists every problem. Fix those values and retry. Don't guess ids that `get_pricing_inputs` didn't return.
- A `get_cost_estimate` row with `missing: true` has an empty field the AWS Pricing Calculator requires; `get_pricing_inputs` marks it `required: true`. Mention it to the user, and fill it in only when the user has given you a value. It doesn't block anything.
- Only change pricing settings the user asked for or agreed to, and say what you changed and how the estimate moved.

### Cost stages
When the user wants the cost of project stages, such as a PoC, an MVP and production, estimate them on one page instead of copying the diagram onto a page per stage.
- **Set up the stages:** call `set_cost_stages` with the stages in order, such as `[{ "name": "PoC", "months": 2 }, { "name": "MVP", "months": 4 }, { "name": "Production" }]`. Give each stage `months` when the user knows how long it lasts, so `get_cost_estimate` can add up each stage's total and the project total. If the page already has pricing settings, pass `currentSettings` with the stage they describe; that's usually production.
- **Price each stage:** pass `stage` (its id or name) to `get_pricing_inputs` and `set_pricing`. A stage starts with the settings of the stage before it, so set the first stage and then change only what grows in each later stage, such as request counts, instance sizes or `count`. `excluded` carries over to later stages like the other settings: for a WAF that only production has, set `excluded: true` on the first stage and `excluded: false` on production. `inherit: true` makes a stage use the previous stage's settings again, inclusion included.
- **Read the estimate:** `get_cost_estimate` with `stage` returns that stage's rows, plus `stages` with every stage's monthly and upfront cost and, for stages with months, its total and the `projectTotal`. Present the stages side by side.
- `get_diagram` shows the page's `stages` and each service's `cost.stages`. Keep both unchanged when you rewrite nodes with `edit_page` or `replace_page`, and change stages only through `set_cost_stages` and `set_pricing`.

## Plan limits
Every team has a plan, shown as `plan` in `list_workspace`. On the Free plan, a team holds at most `diagramLimit` diagrams across all its projects, each of its diagrams holds at most `shapeLimit` nodes across all its pages (arrows don't count), and each person can own a limited number of Free shared teams. On every plan but Enterprise, a team also holds a limited amount of uploaded images. Pages don't count toward `diagramLimit`. `create_diagram`, `duplicate_diagram`, `move_diagram` into another team, `create_team`, page writes that copy images from another diagram and writes that would give a diagram more than `shapeLimit` nodes fail with a plan-limit error; the error says which limit and ends with a link where the team's owner can change the plan. On the Individual and Team plans, upgrading doesn't add image storage, so the image limit error suggests removing images instead and has no link.
- Tell the user which limit the team reached and pass on the link from the error, if it has one.
- Don't retry the call, and don't delete, overwrite or replace diagrams, pages or nodes to make room. Let the user decide: upgrade, make room themselves, or put the diagram somewhere else.
- If `list_workspace` already shows a Free team at its `diagramLimit`, say so before drawing a new diagram for it, instead of finding out when `create_diagram` fails.
- On a Free team, keep each diagram within `shapeLimit` nodes, counting every page. When a request needs more, say so before drawing and ask whether to simplify it or upgrade, instead of finding out when a write fails.

A Free team can also hold more than its limits allow, for example after its paid plan ended. Then part of it is read-only: `list_workspace` marks diagrams nobody can edit with `locked: true`, and a team where the user can only view the diagrams with `viewOnly: true`. `get_diagram` reports `role: "viewer"` for both.
- Read, explain and render these diagrams as usual, but don't try to change them. Writes fail with an error that says the diagram is read-only or that the user can only view the team's diagrams.
- In a `viewOnly` team, don't create, rename, move, delete or share projects or diagrams either.
- Tell the user why, and that the diagram can be edited again once the team is back within the Free plan's limits or its owner upgrades it.
- Don't work around it: don't retry, and don't delete, move or copy diagrams to make room or to edit a copy instead, unless the user asks for that.

## Synced pages
A page can be synced from a JSON file in a GitHub, GitLab or Bitbucket repository. `get_diagram` then gives it a `source` with the `provider` (`github`, `gitlab` or `bitbucket`), `repository`, `branch`, `path`, `url` and the sync `status`. Lineout keeps the page in step with the file, so it's read-only: `edit_page`, `replace_page` and `set_pricing` refuse it.
- To change a synced page, change its file, a single page `{ "nodes": [...], "arrows": [...] }` in the same format as above. If the file is in the repository you're working in, edit it there, check it as described in "Image files", and tell the user when the page will update: for GitHub, once the change is pushed to `branch`; for GitLab and Bitbucket, right after that push if the repository has Lineout's webhook, and otherwise the next time someone opens the diagram or clicks Sync now.
- If the file isn't in your working directory, tell the user which file to change instead of editing the page. Don't copy the page into a new one to change it unless the user asks. The user can also unlink the page in the editor, which makes it an ordinary page again.
- A `status` of `failed` or `paused` comes with an `error` that says why the page isn't updating. Pass it on. A `failed` file usually needs its JSON fixed at the path the error names.

## Locked pages
People can lock a page in the editor so nobody changes it by accident. `get_diagram` marks it `locked: true`, and `edit_page`, `replace_page` and `set_pricing` refuse it. Leave it unchanged: tell the user it's locked and that they can unlock it with Unlock page in the page's tab menu. Don't copy it into a new page to change it unless the user asks.

## Uploaded images
People can upload pictures onto the canvas in the editor. They're image nodes, `{ "type": "image", "image": "<image id>", ... }`, with a required `size` and no `icon`.
- You can move, resize or remove image nodes and give them a `label` as a caption. Keep the width-to-height ratio when resizing, and copy `image` values unchanged.
- You can't upload images or make up image ids. If the user wants a picture on the canvas, ask them to drop or paste it in the editor.
- Writing image nodes from another diagram the user can open, for example when copying a page across, copies those images into the target diagram. On every plan but Enterprise that counts toward the team's image storage; see "Plan limits".
- The CLI draws image nodes as placeholders, because it can't fetch the pictures.

## Safety
- Page edits appear live for everyone who has the diagram open, and nobody can undo them from the editor. Prefer small `edit_page` calls over rewrites.
- Ask before `delete_diagram`, `delete_project`, `delete_page`, `revoke_share_link` and `replace_page` on a page that already has content, and say what will be lost.
- `share_diagram` creates a link that lets anyone who signs in view or edit the diagram. Only create or change one when the user asks.
- If `get_diagram` reports `role: "viewer"`, you can read the diagram but not change it. Say so instead of attempting writes.
- Only a team's owner and admins can rename it. Members, invites, roles and billing are managed in Lineout's settings, not through these tools, so point the user there.
