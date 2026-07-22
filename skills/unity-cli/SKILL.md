---
name: unity-cli
description: Drive Unity from the terminal with the official `unity` CLI — install/manage editors, create projects, open projects, run builds and tests in batch mode. Use when the user wants to create, build, test, or manage Unity projects from the command line, CI/CD Unity pipelines, or asks about Unity Hub headless / batch-mode workflows.
---

# Unity CLI

The official Unity CLI (`unity`) is a standalone binary (experimental/beta) that supersedes Unity Hub's `--headless` mode. It manages editor installs, projects, builds, and tests from the terminal. For anything it doesn't cover, fall back to the classic editor batch-mode arguments (see "Raw editor batch mode" below).

## Setup / verify

```bash
unity --version   # if missing, PATH may need: export PATH="$HOME/.unity/bin:$PATH"
```

Install if absent:

```bash
# macOS / Linux
curl -fsSL https://public-cdn.cloud.unity3d.com/hub/prod/cli/install.sh | UNITY_CLI_CHANNEL=beta bash
# Windows (PowerShell)
$env:UNITY_CLI_CHANNEL='beta'; irm https://public-cdn.cloud.unity3d.com/hub/prod/cli/install.ps1 | iex
```

`unity upgrade` self-updates (`--changelog` previews, `--target <version>` pins). `unity doctor` prints environment diagnostics. `unity changelog` shows what the installed CLI version supports — the CLI evolves fast (beta releases every few weeks) and ships more commands than the docs pages list, so **treat `unity <cmd> --help` as the authority** for the installed version.

## Global flags (work on every command)

- `--format <human|json|tsv|ndjson>` — use `--format json` when parsing output programmatically. JSON responses use a standard envelope: `{success, command, data, errors, warnings}`. NDJSON streams progress events (with a `phase: download|install` field on installs).
- `--non-interactive` — disable prompts; always pass this in scripts/CI
- `--no-banner`, `--quiet`, `--verbose`
- Exit codes: 0 success, 1 error (details on stderr), 130 user-cancelled
- `unity completion <bash|zsh|fish|powershell>` prints shell completions

## Editors

```bash
unity editors -i                        # list installed editors
unity editors -r                        # list available releases
unity editors default 6000.3.7f1        # set default editor
unity editors add /Applications/Unity/Hub/Editor/<ver>/Unity.app   # register existing install
unity install lts                       # install newest LTS (aliases: latest, lts, 6, 2022, ...)
unity install 6000.3.7f1 -m android --cm   # with modules (+child modules e.g. SDK/NDK)
unity install 6000.3.7f1 -m android ios --accept-eula --yes   # fully unattended (CI): no prompts
unity modules list 6000.3.7f1           # module IDs available for a version
unity install-modules -e 6000.3.7f1 -m ios webgl   # add modules later; -l lists available
unity uninstall 6000.3.7f1
unity install-path                      # show/set editor install directory
unity editors path 6000.3.7f1          # print an editor's install dir (use to locate the binary for raw batch mode)
unity install lts --dry-run             # preview download without installing
unity install lts --resume              # resume an interrupted download
```

## Projects

```bash
unity projects list                                    # Hub registry
unity projects new MyGame --path ~/src --editor-version lts --template com.unity.template.3d   # CI-friendly create
unity projects add ./ExistingProject                   # register in Hub
unity projects info                                    # details for current dir project
unity projects require                                 # assert/install the editor version the project needs
unity projects upgrade . --editor-version 6000.3.7f1   # migrate project to another editor
unity open ./MyProject                                 # open in correct editor version (GUI)
unity open "My Game*"                                  # fuzzy/glob match against Hub registry
unity projects clone                                   # clone a repo (GitHub/GitLab/VCS) and register its project
unity projects link / unlink                           # connect project to Unity Cloud or version control
```

Older CLI builds may lack `projects new`; then create a project with raw batch mode:
`<editor-binary> -batchmode -quit -nographics -createProject /path/to/project -logFile create.log`

## Build

`unity build` spawns the editor in batch mode and streams the log. **Unity has no built-in CLI build — you must provide a static C# method** via `--execute-method`:

```bash
unity build . --target StandaloneOSX --execute-method Builder.PerformBuild -o Builds/Game.app
unity build ./MyGame --target Android --execute-method Builder.AndroidBuild \
  --android-export-type aab --output-path ./out/app.aab
unity build . --target WebGL --execute-method Builder.WebGLBuild --allow-install
```

Key flags: `--target` (StandaloneOSX/StandaloneWindows64/Android/iOS/WebGL…), `-o/--output-path` (forwarded as `-buildOutput`; your method must honor it), `-l/--log-file`, `--editor-version`, `--allow-install` (auto-install missing editor), `--allow-dirty-build` (skip the uncommitted-changes guard — builds fail on a dirty git tree without it), `--no-tail`, Android signing via `--android-keystore-base64/-password`, `--android-key-alias`.

Minimal build method to put in `Assets/Editor/Builder.cs`:

```csharp
using UnityEditor;
using UnityEngine;

public static class Builder
{
    public static void PerformBuild()
    {
        var report = BuildPipeline.BuildPlayer(new BuildPlayerOptions
        {
            scenes = new[] { "Assets/Scenes/Main.unity" },   // or read EditorBuildSettings.scenes
            locationPathName = GetArg("-buildOutput") ?? "Builds/Game.app",
            target = EditorUserBuildSettings.activeBuildTarget,
        });
        if (report.summary.result != UnityEditor.Build.Reporting.BuildResult.Succeeded)
            EditorApplication.Exit(1);
    }

    static string GetArg(string name)
    {
        var args = System.Environment.GetCommandLineArgs();
        for (int i = 0; i < args.Length - 1; i++)
            if (args[i] == name) return args[i + 1];
        return null;
    }
}
```

## Test

```bash
unity test                                   # current dir, default test platform
unity test . --mode EditMode                 # or PlayMode
unity test . --filter "MyNamespace.MyTests" --output ./results.xml   # NUnit XML report
unity test . --timeout 1800 -- -nographics   # kill hung editor; args after -- go to Unity
```

## Run arbitrary batch-mode commands

`unity run` picks the right editor from `ProjectVersion.txt` and forwards args after `--`:

```bash
unity run . -- -executeMethod ProjectSetup.Setup -quit -nographics
unity run . --allow-install -- -logFile ./setup.log -quit
```

## Live editor & Player scripting (Pipeline package)

The `com.unity.pipeline` package (experimental, **requires Unity 6.0 LTS or newer**) turns a *running* editor — or a development Player build — into a local API the CLI can drive. This is the primary way for an agent to observe a live project, act on it, and verify the result without edit/recompile/relaunch cycles.

```bash
unity pipeline install                  # add the Pipeline package to a project (pipe = alias)
unity pipeline list                     # editor instances + Pipeline status
unity status                            # live state of connected editors (port, project, PID, state)
unity command                           # no args: list commands the connected editor exposes (self-describing)
unity command <name> [args...]          # execute a command (aliases: cmd, request); --timeout <s>, default 30
unity command --runtime <player-exec>   # target a running dev Player build instead of the editor
```

**`eval` — live C# REPL** (a Pipeline-registered command, so it won't appear in `unity --help`):

```bash
unity command eval "return Application.version;"
unity command eval "return UnityEditor.EditorApplication.isPlaying;"
unity command eval_file path/to/script.cs
```

Compiled with Roslyn, runs on the editor's main thread with full engine/editor API access, answers in milliseconds — use it to inspect scenes, toggle components, enter Play mode, and verify fixes live. Gated behind a security token; the Player runtime API is localhost-only, off by default, and for dev/QA builds only.

**Custom commands**: any static method in the project becomes a CLI command via attributes — no registration step:

```csharp
using Unity.Pipeline.Commands;

public static class MyPipelineCommands
{
    [CliCommand("greet", "Log a greeting and return its length")]
    public static int Greet([CliArg("name", "Who to greet", Required = true)] string name)
    {
        UnityEngine.Debug.Log($"Hello, {name}!");
        return name.Length;
    }
}
// then: unity command greet --name World
```

## CI / auth / licensing

- `unity auth login|status|logout` — interactive, browser-based account sign-in
- **Service accounts for headless CI**: set `UNITY_SERVICE_ACCOUNT_ID` and `UNITY_SERVICE_ACCOUNT_SECRET` env vars — no browser needed
- `unity license list|status|activate|return` — manage licenses on the machine; `unity license server` for floating license servers
- `unity cloud status` / `unity cloud org|project` — Unity Cloud organizations and projects

## Other useful commands

- `unity templates list` / `templates create <project>` — project templates
- `unity mcp` — MCP server/client config for Unity Editor
- `unity logs` — tail the CLI/Hub log file
- `unity shell` — REPL that keeps one warm process for many commands
- Plugin system: any `unity-<name>` executable on PATH becomes `unity <name>`

## Raw editor batch mode (fallback)

When the CLI is unavailable or a task isn't covered, invoke the editor binary directly:

```bash
# macOS binary path pattern:
/Applications/Unity/Hub/Editor/<version>/Unity.app/Contents/MacOS/Unity \
  -batchmode -quit -nographics -projectPath <path> \
  -executeMethod <Class.Method> -logFile <log-path>
```

Gotchas that apply to both CLI and raw batch mode:

- Every batch invocation cold-starts the editor and reimports changed assets — expect 1–5+ min; set generous timeouts and write a `-logFile`, then grep it for `error CS` (compile failures) and your own `Debug.Log` markers.
- Only one editor instance can hold a project open; a running GUI editor blocks batch commands on the same project (lock is `Temp/UnityLockfile`).
- `-quit` is required for one-shot commands or the process never exits; omit it only for commands that manage their own exit (e.g. test runs, `EditorApplication.Exit`).
- Scripts compile before `-executeMethod` runs — a compile error aborts the method silently except in the log.
- Signal failure from C# with `EditorApplication.Exit(nonzero)` so CI sees a non-zero exit code.
