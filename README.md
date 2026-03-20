# Babel3 Plugin Marketplace

Curated plugins for Claude Code, reviewed for quality, accuracy, and safety.

## Installation

```bash
# Add the Babel3 marketplace
claude plugin marketplace add babel3-com/b3-plugins

# Install a plugin
claude plugin install b3@b3-plugins
```

Or browse available plugins with `claude plugin marketplace list`.

## Available Plugins

### plugins/

Plugins developed and maintained by Babel3.

| Plugin | Description |
|--------|-------------|
| [b3](plugins/b3/) | Voice I/O, inter-agent messaging, browser control, and open-source codebase guide. 20 MCP tools. |
| *Your plugin here* | [Submit a PR](#submitting-a-plugin) to list your plugin in the marketplace. |

### external_plugins/

Third-party plugins reviewed and approved for the marketplace. External plugins follow the same directory structure as first-party plugins. *(Coming soon)*

## Plugin Quality Standards

Every plugin in this marketplace meets the [Taste Manifesto](https://babel3.com/blog/taste-manifesto) quality standards:

1. **Invisibility** — A plugin succeeds when you forget it's loaded.
2. **Absence Test** — Remove the plugin. If you can't tell the difference, it was dead weight.
3. **Trigger Precision** — Skills fire when they should. Never when they shouldn't.
4. **Progressive Disclosure** — Essential information first. Details on demand.

Plugins are reviewed by the Babel3 Skill Master before listing. Every skill is tested against real workloads. Every MCP tool is verified against the actual codebase.

## Structure

```
plugins/          — Babel3-maintained plugins
  b3/             — The B3 plugin (voice, hive, codebase skill)
external_plugins/ — Third-party reviewed plugins (coming soon)
```

Each plugin follows the standard Claude Code plugin structure:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json      # Plugin manifest (required)
├── .mcp.json            # MCP server configuration (optional)
├── skills/              # Skills (optional)
│   └── skill-name/
│       ├── SKILL.md     # Skill definition
│       ├── references/  # Detailed reference docs
│       └── examples/    # Working examples
├── agents/              # Agent definitions (optional)
├── hooks/               # Event hooks (optional)
└── commands/            # Slash commands (optional)
```

## Submitting a Plugin

Want your plugin in the Babel3 marketplace? We review every submission for quality, accuracy, and safety.

**Requirements:**
- Plugin follows the standard Claude Code structure
- Skills have specific trigger descriptions (not vague or overly broad)
- All file references resolve correctly
- MCP tools have complete parameter documentation
- No hardcoded credentials or paths

**Process:**
1. Build your plugin following the structure above
2. Test locally with `claude --plugin-dir /path/to/your-plugin`
3. Submit a pull request to this repository under `external_plugins/`
4. We review against the quality standards above
5. Once approved, your plugin is available to all marketplace users

After listing, plugin authors are responsible for maintaining accuracy. Plugins that fail verification against new releases may be temporarily delisted until updated.

## Patent Notice

Babel3 includes patent-pending technology (US 64/010,742, US 64/010,891, US 64/011,207) licensed under Apache 2.0 with a royalty-free patent grant. The patents protect the ecosystem, not restrict it — using this software grants you a royalty-free license to the covered patents.

## License

Apache-2.0. See [LICENSE](../LICENSE).
