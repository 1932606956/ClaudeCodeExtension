# Repository Guidelines

## Project Overview

ClaudeCodeExtension is a Visual Studio 2022/2026 extension (VSIX) that runs AI coding-agent CLIs inside the IDE. It targets **.NET Framework 4.7.2**, is written in C#/WPF, and supports Claude Code (Windows and WSL), OpenAI Codex, Cursor Agent, OpenCode, Devin, PI, OMP (oh-my-pi), Google Antigravity, Reasonix, Qwen Code, and Grok.

There are two presentation modes:

- **Native mode** — the default (`ClaudeCodeSettings.UseNativeMode = true`): a chat tab driven by each CLI's headless JSON/stdio protocol.
- **Terminal mode** — an embedded `conhost.exe` or Windows Terminal window hosted through Win32 `SetParent`, used when native mode is off or the agent has no structured channel (currently Reasonix).

The extension is client-only; settings persist to `%LocalAppData%\ClaudeCodeExtension\claudecode-settings.json`. Licensed MIT.

## Architecture & Data Flow

### Hosting and state ownership

- `Package/ClaudeCodeExtensionPackage.cs` is an `AsyncPackage` that auto-loads with a solution, registers commands/tool windows, and schedules the panel's background load.
- `ToolWindows/` contains the VS tool-window frames: main panel, diff viewer, detached terminal, usage WebView, per-chat document tab, and the multi-chat container.
- `Controls/ClaudeCodeControl.*.cs` is **one `partial class ClaudeCodeControl`** split by feature (~43 files, e.g. `ClaudeCodeControl.NativeMode.cs`, `.NativeChat.cs`, `.Terminal.cs`, `.Diff.cs`, `.Settings.cs`). `UI/ClaudeCodeControl.xaml` is its view. The control owns settings, provider selection, the terminal or native session, prompt sending, and diff integration.
- There is **no DI container**. Partials share private fields declared on the control; each parallel chat tab owns a `NativeChatSessionState` (`Models/NativeChatSessionState.cs`) with its own `IAgentSession` and event queue.

### Agent abstraction

- `Agents/IAgentSession.cs` is the common contract: `StartAsync`, `SendAsync`, `InterruptAsync`, `Received`, `SessionId`, `ResumableSessionId`, `Model`, `SupportsInterrupt`, `SupportsStreaming`, `IsBusy`.
- `Agents/AgentEvent.cs` normalizes all providers onto one flat event model: `SessionStarted`, `AssistantText`, `Thinking`, `ToolCallStarted/Completed`, `PermissionRequested`, `InteractionRequested`, `TurnStarted/Completed`, `UsageUpdated`, `RateLimitUpdated`, `SlashCommandsUpdated`, `SessionError`.
- Adapters:
  - Claude Code: `ClaudeStreamJsonSession` + `ClaudeStreamParser` (bidirectional stream-json).
  - OpenCode, Devin, Devin native, Qwen Code, Grok (and Reasonix, product-disabled): `AcpSession` (JSON-RPC/NDJSON).
  - Codex and Cursor Agent: `OneShotResumeSession` with `CodexExecProtocol` or `CursorAgentProtocol` (process per turn + resume id).
  - PI and OMP: `PiRpcSession`; OMP publishes `available_commands_update` for the "/" picker. Antigravity: `PrintModeSession` (whole answer, no streaming).
- `Agents/JsonLineProcessHost.cs` owns stdio plumbing, line reading, stderr draining, and process-tree teardown; `ClaudeCommandBuilder` / `AcpCommandBuilder` build argv.
- `Agents/` must not reference WPF or the Visual Studio SDK — this is what keeps protocol/parsing code unit-testable.

### End-to-end data flow

1. VS loads the package → `ClaudeCodeToolWindow` creates `ClaudeCodeControl` → `LoadSettings` resolves the workspace directory.
2. `StartNativeModeCoreAsync` (or `InitializeTerminalAsync`) selects a mode. `CreateAgentSession` maps `AiProvider` to an adapter; unsupported/failed starts fall back to the terminal with an info bar.
3. User input (prompt box, `@` mentions, attachments, build/runtime errors, follow-ups) goes through **`SendTextToAgentAsync`**.
4. Native path: `IAgentSession.SendAsync` writes JSON to CLI stdin; the CLI streams NDJSON to stdout.
5. Adapter parses lines into `AgentEvent`s, which are queued (`ConcurrentQueue`) and drained FIFO onto the UI thread; `ApplyAgentEvent` appends text/tool/permission/question rows to the transcript.
6. Turn end replaces streamed text with rendered Markdown (`UI/MarkdownFlowRenderer.cs`) and records usage/cost.

### Diff and review flow

- `Diff/FileChangeTracker.cs` + `DiffComputer.cs` (DiffPlex) build the uncommitted-changes view.
- `Diff/PendingReviewTracker.cs` snapshots a git baseline at prompt time and reconciles changed files per turn; `GitPendingReviewSource` implements `IPendingReviewSource`. Keep/Undo rewrites files and open documents; pending review is in-memory only.

## Key Directories

| Path | Purpose |
|------|---------|
| `Agents/` | Agent session adapters, command builders, JSON-line process host, pure parsers. No UI/VS SDK. |
| `Controls/` | Feature partials of `ClaudeCodeControl`; add new features as `ClaudeCodeControl.<Feature>.cs`. |
| `UI/` | XAML views and pure view-model/helper classes (transcript, composer, Markdown, localization). |
| `ToolWindows/` | VS tool-window frames and `IChatTabHost` for per-tab vs container hosting. |
| `Package/` | VS package entry point and solution/folder event handling. |
| `Models/` | `ClaudeCodeModels.cs` (enums, `ClaudeCodeSettings`) and `NativeChatSessionState.cs`. |
| `Diff/` | File change tracking, diff computation, pending-review baselines. |
| `Tests/` | MSTest unit suite (86 files, ~1,100 tests). SDK-style project. |
| `lang/` | 19 translation JSON files keyed by English UI text; English has no file. |
| `docs/` | `ARCHITECTURE.md` (per-file gotchas), `SKILLS.md` (task recipes), `PERFORMANCE.md`, `AGENTS.md` (short brief). |
| `tools/repo-watcher/` | Dev-only scheduled PR/issue watcher; not part of the shipped extension. |

## Development Commands

Prerequisites: Windows 11 and Visual Studio 2022 (17.x) or 2026 (18.x) with the VS SDK/MSBuild components. Scripts probe VS 2026 first, then VS 2022, and any edition.

```bat
REM Release build (VSIX lands in bin\Release\)
"C:\Program Files\Microsoft Visual Studio\18\Enterprise\MSBuild\Current\Bin\MSBuild.exe" ClaudeCodeExtension.sln -p:Configuration=Release -v:minimal

REM Debug build
"C:\Program Files\Microsoft Visual Studio\18\Enterprise\MSBuild\Current\Bin\MSBuild.exe" ClaudeCodeExtension.sln -p:Configuration=Debug -v:minimal
```

Use MSBuild from a Visual Studio install, **not `dotnet build`**: the VSIX is produced by `Microsoft.VsSDK.targets` / `Microsoft.VSSDK.BuildTools`.

```bat
REM Build Tests\ (Debug) and run the full unit suite via vstest.console.exe
.\test.cmd

REM Build + deploy into the experimental hive (close any Exp instance first)
.\deploy-exp.cmd
.\deploy-exp.cmd -release
.\deploy-exp.cmd -run
.\deploy-exp.cmd -run -open path\to\Other.sln
```

`test.cmd` needs no running VS. For a single test class after the first build:

```bat
"C:\Program Files\Microsoft Visual Studio\18\Enterprise\Common7\IDE\CommonExtensions\Microsoft\TestWindow\vstest.console.exe" Tests\bin\Debug\net472\ClaudeCodeExtension.Tests.dll /TestCaseFilter:"FullyQualifiedName~DiffComputerTests"
```

`publish.cmd` is the maintainer-only marketplace publish (runs `test.cmd`, Release rebuild, `VsixPublisher.exe` + `publishManifest.json`). It is gitignored and may be absent from a clone; `SKIP_TESTS=1` bypasses the test gate.

F5 / Ctrl+F5 inside Visual Studio builds and deploys to the `/rootsuffix Exp` hive automatically.

## Code Conventions & Common Patterns

- **File headers**: every `.cs` file starts with the copyright banner (author Daniel Liedke, 2026, purpose line). Match the existing block.
- **Namespaces**: `ClaudeCodeVS` for controls/models/UI, `ClaudeCodeVS.Agents` for adapters, `ClaudeCodeVS.Diff`, and `ClaudeCodeExtension` only for the package.
- **Naming**: PascalCase public members, `_camelCase` private fields, camelCase locals. Built-in type keywords (`string`, `bool`).
- **Partial classes**: one feature per `Controls/ClaudeCodeControl.<Feature>.cs`; do not create a second state owner. Huge partials (`Terminal`, `NativeMode`, `NativeChat`, `ProviderManagement`) have matching `docs/ARCHITECTURE.md` sections — read the section before editing.
- **Adding files**: the VSIX csproj is old-style and lists files explicitly. Add `<Compile Include="...">`; for XAML add `<Page Include="UI\Foo.xaml">` plus `<Compile Include="UI\Foo.xaml.cs"><DependentUpon>Foo.xaml</DependentUpon></Compile>`. The `Tests` csproj is SDK-style and globs automatically.
- **Agent code**: adapters catch transport failures and emit `AgentEventKind.SessionError` instead of throwing into the UI. Keep stdin open between turns and drain stderr or the child blocks.
- **Async/threading**: UI thread via `ThreadHelper.ThrowIfNotOnUIThread()` / `SwitchToMainThreadAsync()`; process/git work on `Task.Run`. Event handlers are often `async void`. Marshal agent events through the per-session UI pump; never touch WPF from the reader thread.
- **Error handling**: `Debug.WriteLine` (compiled out in Release) plus `%LocalAppData%\ClaudeCodeExtension\terminal-launch.log` for launch diagnostics. User-facing dialogs use `MessageBox` or `VsShellUtilities.ShowMessageBox`.
- **Localization**: new UI text must use `L("...")`, `N("...")`, or `{local:Loc '...'}` and be added to every `lang/*.json` in the same change; `test.cmd` fails on missing or orphaned keys. Never translate agent output, CLI commands, or model/tool names.
- **Settings**: mutate `ClaudeCodeSettings`, then save; writes merge with the on-disk file so multiple VS windows do not clobber provider/model/effort choices.
- **Stable ordinals**: never renumber `AiProvider` values; ordinal 6 is retired and `QwenCode` is 13, `Grok` is 14.
- **Docs stay in sync**: a behavior change in a file indexed by `CLAUDE.md` requires updating its section in `docs/ARCHITECTURE.md`; step-by-step recipes (new provider, setting, UI file, localization, release) live in `docs/SKILLS.md`.

Mandatory cross-cutting rules:

1. Send agent-bound text through `SendTextToAgentAsync`, not `SendTextToTerminalAsync`; console-only traffic is slash commands, CLI updates, and plugin installs.
2. Provider-dependent UI reads `_currentRunningProvider` while a session/terminal is alive; fall back to `_settings.SelectedProvider` only before launch.
3. Focus the embedded terminal only via `FocusTerminalForInputAsync()` / `FocusTerminalForInput()` / `FocusTerminalWindow()`; hook callbacks stay Win32-only.
4. Do not bump `Newtonsoft.Json` from **13.0.3** — it must match the build Visual Studio loads (issue #112).
5. Performance constants (replay limits, throttles, render caps) change only with a new dated measurement in `docs/PERFORMANCE.md`.

**Every code change is a release**: bump the major version only (minor stays `.0`) in `Properties/AssemblyInfo.cs` (`AssemblyVersion` and `AssemblyFileVersion`), `source.extension.vsixmanifest` (`<Identity Version>`), and add a `### Version X.0` entry at the top of README's Version History. `VersionConsistencyTests` enforces all three. Release notes are user-visible only; keep other README edits to one line.

## Important Files

| File | Why it matters |
|------|----------------|
| `Package/ClaudeCodeExtensionPackage.cs` | VS package entry point, command registration, panel background load. |
| `Controls/ClaudeCodeControl.cs` | Core control: initialization, settings load, lifecycle, shared fields. |
| `Controls/ClaudeCodeControl.NativeMode.cs` | `IAgentSession` creation/start, event pump, send/interrupt, terminal fallback. |
| `Controls/ClaudeCodeControl.NativeChat.cs` | Chat tab/composer, model/effort/permission switching, parallel chats. |
| `Agents/IAgentSession.cs`, `Agents/AgentEvent.cs` | The adapter contract and normalized event model. |
| `Agents/JsonLineProcessHost.cs` | Shared stdio/process plumbing for all adapters. |
| `UI/ClaudeCodeControl.xaml`, `UI/ChatTranscriptView.xaml` | Main panel and native-chat transcript views. |
| `Models/ClaudeCodeModels.cs` | `AiProvider`, model/effort enums, `ClaudeCodeSettings`. |
| `Diff/PendingReviewTracker.cs` | Per-turn Keep/Undo baselines. |
| `ClaudeCodeExtension.csproj` | Old-style VSIX project; explicit file lists and VSSDK settings. |
| `source.extension.vsixmanifest` | VSIX identity, version, install targets. |
| `test.cmd`, `deploy-exp.cmd` | Canonical test and experimental-deploy entry points. |
| `CLAUDE.md`, `docs/ARCHITECTURE.md`, `docs/SKILLS.md` | Deep rules, per-file gotchas, and step-by-step recipes. |

## Runtime/Tooling Preferences

- **Windows-only**, .NET Framework 4.7.2, WPF + WinForms + UI Automation; VSIX targets VS 17.0–18.x on amd64 and arm64.
- **MSBuild from Visual Studio** is the build tool; there is no `global.json`, `Directory.Build.props`, or pinned .NET SDK.
- **NuGet** via `PackageReference`: `Microsoft.VisualStudio.SDK` 17.14.40265, `Microsoft.VSSDK.BuildTools` 18.5.40034, `Newtonsoft.Json` 13.0.3, `DiffPlex` 1.9.0, `Microsoft.Web.WebView2` 1.0.4078.44.
- **No CI workflows, `.editorconfig`, analyzers, StyleCop, or `TreatWarningsAsErrors`** — style is convention-enforced. `WarningLevel` is 4.
- **No Git LFS.** `publish.cmd` is deliberately gitignored (contains a PAT); never commit it.
- From Git Bash, scripts must use `%SystemRoot%\System32\find.exe` to avoid GNU `find`; from WSL, invoke MSBuild through `powershell.exe`.

## Testing & QA

- **Framework**: MSTest 3.6.4 (`MSTest.TestFramework` + `MSTest.TestAdapter`) with `Microsoft.NET.Test.Sdk` 17.12.0, targeting `net472`. Run everything with `.\test.cmd`; it builds Debug and invokes `vstest.console.exe`.
- **Layout**: one `<Area>Tests.cs` per area in namespace `ClaudeCodeExtension.Tests`; `[TestClass]` / `[TestMethod]`, descriptive `Subject_Scenario_Expected` names, `[DataRow]` for tables. `InternalsVisibleTo("ClaudeCodeExtension.Tests")` exposes internal helpers.
- **Isolation**: tests are pure unit tests — parsers, formatters, command builders, path/session logic, and view-model math. Protocol tests feed recorded JSON lines into parsers; `PendingReviewTrackerTests` uses `FakeRepository : IPendingReviewSource`; `DevinUsageTests` uses a `StubHandler : HttpMessageHandler`. No real CLI, git remote, or VS shell.
- **Source-level guard tests** read files as text and assert structure: `VersionConsistencyTests`, `PackageVersionGuardTests`, `LocalizationTests`, `NativeSessionLifecycleTests`, `ResumedTranscriptReplayTests`.
- **Manual QA**: terminal embedding (`SetParent`), provider round-trip, settings dialog, and anything requiring a running VS are exercised via F5 / `deploy-exp.cmd` in the Exp hive.
- **Expectations**: no coverage threshold or `[Ignore]` culture. Add a pure unit test for new logic; add or update a guard test for metadata/version/localization rules. Keep `test.cmd` green — `publish.cmd` gates releases on it.
