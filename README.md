# Lineout plugins for Claude Code

Claude Code plugins for [Lineout](https://lineout.app), a collaborative editor for cloud architecture diagrams.

```bash
claude plugin marketplace add lineout-app/plugins
claude plugin install lineout@lineout --config lineout_url=https://lineout.app
```

Then run `/mcp` in Claude Code, pick `plugin:lineout:lineout` and authenticate. See [plugins/lineout](plugins/lineout/README.md) for details.

To render diagrams to images locally, the skill uses the `lineout` command from [`lineout-cli`](https://pypi.org/project/lineout-cli/):

```bash
uv tool install "lineout-cli[jpg]"
```

This repository is published from the Lineout app repository. Changes made here directly will be overwritten.
