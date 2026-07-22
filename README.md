# unity-cli — Claude Code plugin

A [Claude Code](https://claude.com/claude-code) plugin that teaches Claude to drive the **Unity game engine from the terminal**:

- Install and manage Unity editors and modules (official `unity` CLI, the successor to Unity Hub's headless mode)
- Create and register projects (`unity projects new`, Hub registry, templates)
- Run **builds** in batch mode — including the mandatory `--execute-method` pattern with a ready-to-paste `Builder.cs`
- Run **EditMode/PlayMode tests** with NUnit XML reports
- Fall back to raw editor batch-mode arguments when the CLI doesn't cover something, with the gotchas that actually bite (asset-import cold starts, `Temp/UnityLockfile`, `-quit` semantics, silent compile-error skips, CI exit codes)

## Install

In Claude Code:

```
/plugin marketplace add cguldogan/unity-cli-skill
/plugin install unity-cli@unity-cli-skill
```

## Use

The skill activates automatically when you ask Claude for Unity-from-the-terminal work, or invoke it explicitly:

```
/unity-cli
```

Example prompts:

- "Create a new Unity project with the 3D template using the latest LTS"
- "Build this Unity project for macOS from the command line"
- "Run this project's PlayMode tests and show me the failures"
- "Install Unity 6 LTS with the Android module"

## Requirements

- Unity CLI (the skill includes install instructions Claude will follow), or any Unity editor install for the raw batch-mode fallback
- macOS/Linux/Windows with a Unity-supported environment

## What's inside

```
.claude-plugin/
  plugin.json          # plugin manifest
  marketplace.json     # lets the repo act as its own marketplace
skills/
  unity-cli/
    SKILL.md           # the skill: command reference + workflows + gotchas
```

Command reference is grounded in Unity CLI `1.0.0-beta.2` `--help` output and the [official Unity CLI docs](https://docs.unity.com/en-us/unity-cli/unity-cli).

## License

MIT
