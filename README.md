# Kestrel Plugin

Kestrel language guidance (a skill) plus the `kestrel-lsp` language-server
configuration, packaged as a plugin for **Claude Code** and **Codex**.

## Install

**Claude Code**

```sh
claude plugin marketplace add kestrellang/kestrel-plugin
claude plugin install kestrel-plugin@kestrel
```

**Codex**

```sh
codex plugin marketplace add kestrellang/kestrel-plugin
codex plugin add kestrel-plugin@kestrel
```

The plugin provides a Kestrel skill (syntax, ownership, package tooling, common
pitfalls) and configures the `kestrel-lsp` language server for `.ks` files.
`kestrel-lsp` must be on your `PATH` — Jessup installs and links it with the
active Kestrel toolchain.

## Layout

This repo is a marketplace hosting a single plugin:

- `.claude-plugin/marketplace.json` — Claude Code marketplace manifest.
- `.agents/plugins/marketplace.json` — Codex marketplace manifest.
- `plugins/kestrel/` — the plugin itself:
  - `.claude-plugin/plugin.json`, `.codex-plugin/plugin.json` — host manifests.
  - `.lsp.json` — `kestrel-lsp` configuration for `.ks` files.
  - `skills/kestrel/SKILL.md` — the Kestrel language guide (source of truth).

## Editor extension (VS Code / Cursor)

The Kestrel editor extension ships as a `.vsix` in the
[`kestrel-vscode` releases](https://github.com/kestrellang/kestrel-vscode/releases/latest).
Download the `.vsix` for your platform and install it:

```sh
code --install-extension path/to/kestrel-<target>.vsix
```

The extension discovers `kestrel-lsp` from `PATH`; Jessup provides it with the
active toolchain.
