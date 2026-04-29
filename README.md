# Kestrel Plugin

Local plugin package for Kestrel language guidance.

## Contents

- `.codex-plugin/plugin.json` declares the Codex plugin metadata and skill path.
- `.claude-plugin/plugin.json` declares the Claude Code plugin metadata and skill path.
- `.lsp.json` configures the `kestrel-lsp` language server for `.ks` files.
- `KESTREL_SKILL.md` is the standalone Kestrel language guide.
- `skills/kestrel/SKILL.md` is the discoverable skill version of the same guide for both plugin hosts.

## Claude Code

Claude Code discovers plugin components from the plugin root. This package uses
the standard `.claude-plugin/plugin.json` manifest and root-level `skills/`
directory, so the skill is installed as a namespaced plugin skill.

The LSP configuration expects `kestrel-lsp` to be available on `PATH`. Jessup
installs and links it with the active Kestrel toolchain.

## VS Code

Install the official Kestrel VS Code extension from the Extensions sidebar by
searching for **Kestrel**, or install it from the command line:

```sh
code --install-extension kestrel-lang.kestrel
```

For a downloaded release asset, install the matching `.vsix` directly:

```sh
code --install-extension path/to/kestrel-<target>.vsix
```

To install the extension from the latest GitHub release:

```sh
target="$(case "$(uname -s)-$(uname -m)" in
  Darwin-arm64) echo darwin-arm64 ;;
  Darwin-x86_64) echo darwin-x64 ;;
  Linux-x86_64) echo linux-x64 ;;
  *) echo unsupported; exit 1 ;;
esac)"
gh release download --repo kestrellang/kestrel-vscode --pattern "kestrel-${target}.vsix" --clobber
code --install-extension "kestrel-${target}.vsix"
```

After installing, open a `.ks` file or a folder containing `flock.toml`. The
extension discovers `kestrel-lsp` from `PATH`; Jessup provides it with the
active toolchain.
