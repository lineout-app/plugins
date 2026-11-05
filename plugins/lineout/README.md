# Lineout plugin for Claude Code

Bundles two things:

- **`lineout` MCP server**: Lineout's `/api/mcp` endpoint. Through it, Claude can list your teams, projects and diagrams, read and edit diagram pages live, manage share links and search the AWS icon set.
- **`diagrams` skill** (`/lineout:diagrams`): how to turn a description or infrastructure code (Terraform, CDK, CloudFormation, Kubernetes, docker-compose) into a well laid out Lineout diagram, and how to edit existing diagrams safely. Claude also uses it automatically when you ask for a diagram.

## Install

```bash
claude plugin marketplace add lineout-app/plugins
claude plugin install lineout@lineout --config lineout_url=https://lineout.app
```

`lineout_url` is the origin of the Lineout deployment the MCP server connects to; it defaults to `https://lineout.app`. Pass it explicitly, because installing from the command line may leave it unset. To work against a local dev server, add the marketplace from a clone of the Lineout app repository (`claude plugin marketplace add ./`) and install with `--config lineout_url=http://localhost:3000`. To switch an existing install, uninstall and install again with the new URL, or run `/plugin configure lineout@lineout` in Claude Code.

Then run `/mcp` in Claude Code, pick `plugin:lineout:lineout` and authenticate. Your browser opens Lineout's consent page. Approved connections are listed, and can be disconnected, under **Settings → Connected apps**. After changing the URL, authenticate again.

To update after a new version is pushed:

```bash
claude plugin marketplace update lineout
claude plugin update lineout@lineout
```

If you added the server by hand before (`claude mcp add ... lineout`), remove it (`claude mcp remove lineout`) so the tools aren't listed twice.

## Develop

```bash
claude --plugin-dir packages/claude-plugin
claude plugin validate packages/claude-plugin --strict
```

The skill lives in `skills/diagrams/`: `SKILL.md` (workflow and safety rules), `layout.md` (spacing, grouping and arrow rules), `rendering.md` (rendering pages to images with the `lineout` CLI from `packages/python`, to check layouts by eye and to produce image files) and `examples/serverless-api.json` (a valid page to copy from).

The skill uses the `lineout` CLI when it's installed. Install it with `uv tool install "lineout-cli[jpg]"`. To develop against local changes instead, run `pnpm build:python` in the app repository, then `uv tool install --editable "./packages/python[jpg]"`. The tool descriptions and the format guide that `get_format_guide` returns live in the app, in `lib/server/mcp/`.
