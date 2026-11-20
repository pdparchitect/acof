# Changelog

All notable changes to Acof, following [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and [Semantic Versioning](https://semver.org/).

## [0.7.0] - 2026-11-20

### Changed

- **Rook is now acof.** The tool, its binary and everything it reads take the new name: the command is `acof`, objectives and the ledger live under `.acof/`, the config under `~/.config/acof/`, and every `ROOK_*` environment variable is now `ACOF_*` (`ACOF_CONFIG`, `ACOF_AGENT_MODEL`, `ACOF_DEFAULT_PROVIDER`, `ACOF_VERSION`, `ACOF_INSTALL_DIR` and the rest). This is a breaking change with no fallback: rename an existing `.rook/` folder, `~/.config/rook/` and any `ROOK_*` variables to keep using them. Releases and the installer come from `pdparchitect/acof`, as `acof-<version>-<os>-<arch>.tar.gz`. The Rook artwork is gone: the README has no banner, the run viewer takes zot's neutral look instead of a red accent, and the desktop image has a plain icon and wallpaper and an unbadged greeting.

- **The README is minimalist; detail moved into `docs/`.** The front page had grown to ~500 lines; it now leads with the essentials - what Acof is, install, a first run, why, safety - and a Documentation section, matching zot's shape. The providers, configuration, objectives/flags, skills, how-it-works, safety and development sections moved into a `docs/` folder, each its own page.

- **Release notes now come from this changelog, matching zot.** The release workflow set `generate_release_notes: true`, so a GitHub release listed raw commit titles and ignored `CHANGELOG.md` entirely - while zot published the curated changelog section as its release body. acof's workflow now extracts the top-most `## [version]` section from `CHANGELOG.md` (the same `awk` zot uses) and publishes it as the release body. The changelog format was already identical; only the workflow differed. Every future release reads like the changelog rather than a commit list.

## [0.6.1] - 2026-08-24

### Changed

- **"Backend" is now "provider", to match zot.** Acof and zot are configured the same way but spoke different dialects: acof called a named model-provider connection a *backend*, zot called it a *provider*, and where zot's connection names its implementation with `driver:`, acof's used `provider:`. They now agree on zot's vocabulary. The config keys `default_backend` → `default_provider` and `backends:` → `providers:`; the inner `provider:` field (the implementation) → `driver:`; the flag `--backend` → `--provider`; and the env var `ACOF_DEFAULT_BACKEND` → `ACOF_DEFAULT_PROVIDER`. This is a breaking config change - a config file or script using the old keys must be updated - but it makes moving between the two tools one vocabulary instead of two.

### Added

- **The engine tuning knobs, and per-model capability overrides, to match zot.** acof's `agent:` config exposed only `model` and `max_iterations`; the rest of the engine's controls were hardcoded or unreachable. It now carries the same knobs zot does - `max_settles`, `max_calls`, `max_time`, `max_tokens`, `max_tool_output`, `max_continuations`, `max_recoveries`, `max_cycles`, `max_empties`, `limit_checkpoints`, `context_strategy`, and the `compact_*` trio - each optional, zero using the engine default, and validated at load. A model entry also takes `context` (correct a too-large assumed context window for a small endpoint) and `vision` (say a model can be shown images, so the agent is offered the view tool - previously acof never enabled vision at all). acof cannot import zot's internal model catalogue, so there is no auto-detection; these are explicit operator overrides, which is the part that matters for custom endpoints.

### Fixed

- **A provider's API key never reaches a custom endpoint it was not written for, and never reaches the agent's shell.** A built-in provider's conventional key (e.g. `OPENAI_API_KEY`) was seeded as its credential purely from its name. Two consequences are now closed: (1) when a built-in provider is pointed at a custom `base_url`, its ambient key is no longer adopted as the connection's credential, so a key scoped to the provider's real host is never forwarded to a URL from the config; and (2) `ScrubProviderSecrets` now unsets every built-in provider's conventional variable by name - not only those whose value was resolved into config - so a provider key exported in the environment is stripped before the agent runs even when that provider has a `base_url` set. For an offensive-security tool whose agent runs commands against targets, both keep a provider credential out of reach of those commands.

## [0.6.0] - 2026-08-24

### Added

- **One-line installer, and an update notice that tells you how to upgrade.** Acof now ships an `install.sh` - published as a release asset - that downloads the latest release for the platform, verifies it against the release checksums, and drops `acof` in `~/.local/bin`: `curl -fsSL https://github.com/pdparchitect/acof/releases/latest/download/install.sh | bash` (pin with `ACOF_VERSION`, relocate with `ACOF_INSTALL_DIR`). The same command upgrades in place, and acof's update notice now prints it - previously the notice said a new version existed and linked the release page but never how to get it, matching zot's notice which already carried its installer line. The installer is the repo's `install.sh`, the single source of truth, not a copy maintained elsewhere.

- **The update check can be turned off.** Acof makes one call per run that is not to the model provider - a GitHub lookup for the latest release - and until now there was no way to stop it, which is the wrong default for the air-gapped and locked-down boxes acof is built to run on. `update_check.disabled: true` in the config, or `ACOF_UPDATE_CHECK_DISABLED=true` (no config file needed, for containers and CI), makes no call at all. This mirrors zot's `update_check` switch.

- **An end-of-run digest with the session id to resume.** A run scattered its record across a few stderr lines and ended with the full-screen viewer's stats disappearing as the terminal restored, and the session log's location was only ever shown as a full path at startup — so "which session do I resume?" meant scrolling back for it. Every run now closes with a compact block: `status`, `session`, a ready-to-copy `resume` command (`acof --resume <id>`), `iterations`, `calls`, `input-tokens`, `output-tokens`, and the ending `message`. It is one row per line, a single-word key then the value — no borders, no ANSI — so it reads at a glance and parses with a single `awk`. Printed to stderr after the viewer restores the screen, so it survives where the alt-screen stats did not, and shown whether the run rendered full-screen or streamed plain.

- **Skills load dynamically from `~/.config/acof/skills`, picked up mid-run.** Acof shipped its skill library compiled into the binary and read nothing else, so a skill was fixed at build time: adding one meant a new release. The embedded set is now the baseline, layered under whatever lives in `~/.config/acof/skills` (`$XDG_CONFIG_HOME/acof/skills`), and that directory is rescanned every iteration rather than once at startup. A `SKILL.md` placed there appears in the agent's available-skills list on its next turn — whether the operator dropped it in or the agent fetched it itself. This is the autonomy path the design was reaching for: the agent can `git clone` a skill collection into that folder with the `shell` tool it already has and read the new skills the same turn, no restart and no operator step. A directory skill wins a name clash with an embedded one, so a downloaded skill can override a shipped default; if the directory turns unreadable mid-run the last good set is kept rather than blanking the agent's skills.

- **Session recording and `--resume`.** Every run now writes a JSON Lines session log under `.acof/sessions/`, so a run can be inspected afterwards and resumed. `acof --resume last` (or an id, or a path) continues the conversation where it left off, so a multi-hour audit that crashed overnight loses nothing. `acof sessions` lists past runs with their status and task. `--no-session` disables recording for a run. The runner uses a composite recorder so both the live status artifacts (for the widget) and the session log (for resume) are driven from one event stream.

- **`--dir` flag.** Point acof at a target tree without `cd`-ing into it: `acof --dir ./target audit.yaml` investigates that directory. This is convenience, not containment - the agent's shell commands retain the process's host permissions.

- **Coverage gate.** `make cover-check` and CI run the same `scripts/coverage.sh` script, failing the build if total statement coverage drops below 75%. The bar is deliberately lower than zot's 90%: acof has thin packages (`cmd`, `buildinfo`, `version`) that are partly untestable wiring, and the agent runner cannot exercise a real model provider in CI. Raise it as coverage grows.

- **AGENTS.md and testing skill.** An `AGENTS.md` agent instruction file and a `.agents/skills/testing-and-coverage/SKILL.md` convention guide, mirroring zot's setup, so an agent working in this repo knows the coverage bar, how tests are written, and what levels to reach for.

- **Objectives, not prompts.** Acof now dispatches from mission files, not command-line prose. An objective is a small YAML file - the durable goal, the success criteria that define "done", and the rules of engagement the work must hold to - and each objective becomes one autonomous run. Objectives live under `.acof/objectives/` and their run records under `.acof/records/`, the same convention zot uses for its work orders. The file is the contract: it can be edited, committed, re-run, and judged against the evidence the run produced. `acof new "the objective"` scaffolds one; a bare `acof` runs every outstanding objective in the dossier, in filename order, skipping what the ledger already records as done; `acof --watch` turns the folder into a drop box.

- **Ledger of completed runs.** A satisfied objective is skipped on re-run, so restarting a batch picks up where things left off. Editing the objective changes its hash and re-queues it. A failed run is never recorded as done, so it runs again next time. Each receipt carries evidence read back from the run's own artifacts - the stop reason, the summary, the iteration count - so a record is proof rather than a claim.

### Removed

- **The 51 embedded security skills, and the third-party license that came with them.** Acof shipped ~1.5 MB of markdown playbooks compiled into every binary - 21k lines of externally-sourced content that could only be updated by cutting a new release, and that dragged a bundled MIT license and a `NOTICE.md` along with it. Both are gone. In their place the binary carries a single **`skill-catalog`** skill: an index that names external skill collections and tells the agent how to fetch them. The catalog points at [Claude-BugHunter](https://github.com/elementalsouls/Claude-BugHunter) (the same content that used to be embedded), and the agent clones it into `~/.config/acof/skills` with the `shell` tool when an objective needs it — where the dynamic loader picks it up on the next turn. The result: a leaner binary, a skill library that grows without a acof release, and no third-party content or license carried in the tool itself. `NOTICE.md` is deleted; the README's skill sections and credits are rewritten to describe fetch-on-demand rather than a bundled library.

### Changed

- **Every skill is read with `read`; the `skill` tool is gone.** An embedded skill used to be served by a dedicated `skill` tool (it had no filesystem path for `read` to open), while a disk skill was read with `read` — two tools for one job, and the `skill` tool's list of valid names was frozen at startup, so it structurally could not name a skill added while the run was going. Embedded skills are now addressed by an `embedded-skill:///<name>` URL that the `read` tool resolves against the in-binary set; the scheme names itself, so the agent infers a skill is embedded from its path without the prompt having to say so. Reading one is now identical to reading a file — same bounded line range, same line-numbered output — and the backstory tells the agent to `read` each skill at the path given in its available-skills entry, embedded or on disk alike.

- **Bumped to zot 0.20.0** (was 0.19.1). Picks up the engine's dynamic skill loading — `Options.Skills` / `ExecuteWithToolsOptions.Skills` are now `func() []Skill`, re-read each iteration — and the `agent.SkillLoader` Acof wires over its skills directory, plus the removal of `SkillsResult.Tool()` in favour of `read` resolving `embedded-skill://` URLs from the `EmbeddedSkills` registry passed to `DefaultToolsFor`. Also absorbs the digest helpers the end-of-run block is built on: `tui.RenderDigest`/`tui.DigestStatus`, the new `InputTokens`/`OutputTokens` on `agent.Summary`, and `agent.SummaryRecorder`/`agent.MultiRecorder`. The session log's `Result` now persists those token totals as well, so a resumed run or an after-the-fact audit of a Acof session sees what it cost, not just its iteration and call counts. Built against the in-repo sibling via a `replace` directive so the shared engine and viewer stay in lockstep.

- **Bumped to zot 0.19.1** (was 0.9.1). Absorbs ten minor versions of upstream improvement: the `agent.Recorder` interface gains `RecordFailure` (so a run killed mid-retry still leaves the failing exchange in the event log), `tui.Run` now returns an `Outcome` (the stop reason and summary the terminal tool carried), and `ExecuteWithToolsOptions` carries new recovery bounds (`MaxCalls`, `MaxRecoveries`, `MaxContinuations`), context-strategy tuning, and the non-interactive contract that keeps an unattended run from stalling on a question. Built against the in-repo sibling via a `replace` directive so the shared engine and viewer stay in lockstep.

- **Authorization scope removed.** The `scope` field is gone from the objective schema, the `--scope`/`--scope-file` flags are gone, and the backstory no longer carries an authorization-boundary section. Acof no longer injects a scope into the agent's system prompt or records one in the run's status artifact. Use the objective's `rules_of_engagement` to constrain how the agent operates.

- **Acof renders through zot's shared viewer (`github.com/openzot/openzot/tui`), themed red.** The hand-rolled event loop that printed activity to stderr is gone; Acof now hands the engine a `tui.Meta` (its own `acof` name in the badge, a red accent, the iteration cap shown as `iter N/10000` progress) and the viewer renders the run. On a terminal that is the full-screen view zot ships; piped, redirected or under `--verbose` it streams the same plain text as before, so logs and CI are unchanged. The semantic status colours (running / done / failed) stay fixed - only the brand accent is Acof's.
- **Run artifacts are written through the engine's `Recorder`, not a private loop.** `status.json` and `events.jsonl` are now produced by an `agent.Recorder` the engine drives from the same event stream that feeds the viewer, so rendering and recording no longer share a switch statement. The on-disk format is unchanged.
- **Bumped to zot 0.9.1.** Absorbs the upstream rename of the system-prompt field (`Backstory` → `Instructions`) and picks up provider-reported token usage, configurable context strategy, and the embeddable, themeable TUI (0.9.1 adds `tui.Meta.AppName`, so the viewer reads as "acof"). Built against the in-repo sibling via a `replace` directive so the shared engine and viewer stay in lockstep.
- `--verbose` now selects plain streaming over the full-screen viewer (it still carries the reasoning tokens); the help text says so.

## [0.4.0] - 2026-08-05

### Features

- Run artifacts: every run now writes its own `status.json` (live state: model, scope, iteration, current tool, timestamps, exit) and `events.jsonl` (an append-only event log) into a per-run directory `<run_dir>/<runid>/`. This is telemetry, separate from the workspace the agent works in, and is what the desktop status widget reads. It is always on, multi-instance safe (`<runid>` is `<timestamp>-<pid>`), and the base directory is `$XDG_STATE_HOME/acof/runs` by default (override with `--run-dir`, `run_dir` in config, or `$ACOF_RUN_DIR`).

### Changed

- **The engine is now [zot](https://github.com/openzot/openzot), running in-process.** Acof drops its hosted agent SDK and no longer talks to a hosted service: the agentic loop, thread management, compaction and loop detection all run inside the binary, straight against a model provider. A run is reproducible offline and depends on nothing staying up but the provider you point it at.
- **Native backends.** Acof targets a provider directly - `zai` (default, running `glm-5.2`), `openai`, `anthropic`, `groq`, `mistral`, `deepseek`, `openrouter`, `together`, `cerebras`, `xai`, `moonshot`, `qwen`, and a local `ollama` - each reading its provider's conventional key. A backend key is written as `api_key`, literal or a `$VAR` reference. A local Ollama is the recommended choice for material that must not leave the machine.
- **Breaking (security):** a released binary no longer reads a `.env` from the working directory. Acof runs shell commands against targets with a provider key in the process, so taking credentials from whatever directory it was pointed at is a liability - a stray committed `.env` in the code under review would otherwise reach the process about to run commands against it. `make dev` (or `go build -tags dev`) still reads it, for local development; the switch is a build tag that defaults to off, and `acof --version` prints which kind you have.
- **Settle mode.** A run now ends only when the agent records an outcome - `_success` with a summary, or `_failure` with a reason - never because its prose sounded conclusive. An unattended security run needs an unambiguous ending.
- `make` prints the available targets instead of assuming `build`, and `make vet` covers both build variants. The desktop image and its Makefile pass provider keys through to a containerised run.

## [0.3.0] - 2026-08-05

### Features

- `acof config` opens the config file in your `$EDITOR`, creating it from a commented template (embedded in the binary) on first run. This is the setup path - choose a backend and model and set your provider key by editing the file. `acof config path` prints the file location.

### Changed

- **Breaking (auth):** the `relay` backend no longer reads a `RELAY_API_KEY` environment variable. Its credential is your own provider key, set per model (`backends.relay.models.<model>.authorization`) or as a backend-level default (`backends.relay.authorization`) in the config file, or inlined into `--model` as `<model>/authorization=<key>`. A relay run with no such key fails with an actionable error rather than falling back to an env var.
- The default model is now `glm-5.2` (was `qwen-3.6-plus`) - a strong open model suited to autonomous security work.
- The example config showcases open security models (`glm-5.2`, `kimi-k3`, `deepseek-v4-flash`).

## [0.2.0] - 2026-08-05

### Features

- Configuration file: Acof now reads a layered configuration - built-in defaults < `~/.config/acof/config.yaml` < `ACOF_*` environment variables < CLI flags. The file path is overridable with `$ACOF_CONFIG` or `--config`, and values may reference secrets with `$VAR`. The file is optional; env vars alone still configure a run. See [configs/acof.example.yaml](configs/acof.example.yaml).
- Backends: a run targets a named **backend** via `--backend` (or `default_backend` in config). Three ship built in - `relay` (CBK Relay, the default), `cbk` (ChatBotKit at `api.cbk.ai`) and `chatbotkit` (ChatBotKit at `api.chatbotkit.com`). Under `backends.<name>.models`, a custom entry can alias a real model id and override `max_iterations`.
- Relay per-model authorization: the CBK Relay authenticates each model with its own provider key, carried inside the model string as `<model>/authorization=<key>` (each model may be a different provider). Acof composes it from a model's `authorization`, a backend-level default, or `$RELAY_API_KEY`; a key already inlined into `--model` is left untouched.
- Secret hygiene: resolved backend credentials - Bearer secrets and provider authorizations, backend-level and per-model - are stripped from the environment before the agent runs, so the commands it executes against a target cannot read them.

### Changed

- **Breaking (auth):** the default backend is now `relay`, whose credential is `RELAY_API_KEY` (your OpenAI/OpenRouter key). To reach ChatBotKit as before, use `--backend cbk` with `CBK_API_SECRET`, or `--backend chatbotkit` with `CHATBOTKIT_API_SECRET`.
- The Go SDK is bumped to `github.com/chatbotkit/go-sdk v0.4.0`.
- `.env` is now loaded only as a convenience for populating the environment; the config file is the primary configuration surface.

## [0.1.2]

- Initial public releases: single self-contained Go binary, embedded security skill library, autonomous agent loop over the ChatBotKit API, and cross-platform release archives.
