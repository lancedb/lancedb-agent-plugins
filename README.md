# LanceDB Agent Plugins

Agent plugins from [LanceDB](https://www.lancedb.com) for building multimodal data pipelines.

## What's included

The **lancedb** plugin helps coding agents write, review, debug, and document LanceDB pipelines — code that works across local LanceDB OSS tables and remote LanceDB Enterprise tables. It guides agents toward idiomatic query/search patterns and LanceDB performance defaults for ingestion, indexing, filtering, and diagnostics, and away from non-portable patterns like full-table materialization.

## Installation

### Codex

```
codex plugin marketplace add lancedb/lancedb-agent-plugins
codex plugin add lancedb@lancedb
```

### Claude Code

```
/plugin marketplace add lancedb/lancedb-agent-plugins
/plugin install lancedb@lancedb
```

### Other tools

The cross-tool [plugins](https://github.com/vercel-labs/plugins) installer works for Cursor, GitHub Copilot CLI, VS Code, and others (as well as Codex and Claude Code):

```
npx plugins add lancedb/lancedb-agent-plugins
```

## Layout

```
.claude-plugin/   Plugin and marketplace manifests
skills/lancedb/   The LanceDB skill: instructions, references, and scripts
assets/           Logos
```

## License

[Apache 2.0](LICENSE)
