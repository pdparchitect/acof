# Objectives

Acof dispatches from mission files, not command-line prose. An objective is a
small YAML file - the durable goal, the success criteria that define "done", and
the rules of engagement the work must hold to - and each objective becomes one
autonomous run.

```bash
export ZAI_API_KEY="sk-..."       # or --provider openai with OPENAI_API_KEY, etc.

# Write an objective, then run it
acof new "Gain access to the target network and map paths to domain admin"
# edit .acof/objectives/gain-access-to-the-target-network-and-map-pa.yaml
# to set the success criteria, then:
acof

# Run a single objective with reasoning streamed to the terminal
acof -v .acof/objectives/firmware-recon.yaml

# Version
acof version
```

`acof new` drops a file under `.acof/objectives/`. Edit it to set the success
criteria, then a bare `acof` runs every outstanding objective in the dossier -
skipping what the ledger already records as done. `--watch` turns the folder
into a drop box.

```bash
acof                                          # run every outstanding objective
acof .acof/objectives/firmware-recon.yaml   # run one by name
acof --watch                                  # drop-box mode
```

A ledger records what has run: a satisfied objective is skipped on re-run;
editing an objective changes its hash and re-queues it; a failed run is never
recorded as done, so it runs again next time. Each receipt carries evidence read
back from the run's own artifacts - the stop reason, the summary, the iteration
count - so a record is proof rather than a claim.

A developer build loads a `.env` from the working directory; a released binary
does not (see [development.md](development.md)).

## Some example objectives

```bash
# Reverse-engineer an entire binary or firmware image - recover structure,
# embedded secrets, and the attack surface it exposes
acof new "Reverse engineer the firmware image in ./firmware.bin: recover embedded credentials, map services, and identify remotely reachable bugs"

# Full engagement: gain a foothold and map how deep access goes
acof new "Gain initial access to the target network from the external surface, then map lateral-movement paths to the domain controller"

# Whole-codebase audit - not one endpoint, the whole project
acof new "Audit the entire ./monorepo for the full OWASP taxonomy: injection, broken access control, auth flaws, crypto misuse, and unsafe deserialization"

# Cloud compromise assessment - find the path from a foothold to data exfiltration
acof new "Assess the AWS environment for paths to privilege escalation; chain IAM, S3 and IMDS exposure into a full data-exfiltration chain"

# External attack-surface mapping
acof new "Map example.com's external surface: subdomains, exposed services, leaked credentials, and anything that should not be internet-facing"

# Smart-contract security review - the whole protocol, not one function
acof new "Audit the Solidity protocol in ./contracts for reentrancy, access-control, oracle manipulation and economic attacks across every contract"
```

## Objective files

An objective is a small YAML file with four fields:

```yaml
# .acof/objectives/firmware-recon.yaml
# acof objective - what to do and what "done" means.

title: Firmware reverse engineering

# The durable goal of the engagement. The objective will not run until this is filled in.
objective: |-
  Reverse engineer the firmware image: recover embedded credentials, map the
  services it exposes, and identify remotely reachable memory-corruption bugs

# The objective is not met until every one of these holds.
success:
  - every credential is extracted and documented with its location and purpose
  - every remote service is enumerated with its protocol and entry point
  - every exploitable bug has a working PoC and impact assessment
  - the engagement report is delivered as the run's outcome

# Rules that hold for the whole run - non-negotiable constraints on how the
# objective may be pursued.
rules_of_engagement:
  - no network access; work entirely offline against the image
  - do not modify the original firmware image
```

The agent's findings stream to **stderr**; with `--verbose`, reasoning tokens
stream to **stdout**. The final report is delivered as the agent's response -
Acof does not write files on its own. If you want the report (or any other
artifact) saved to disk, ask for it in the objective and the agent will use its
`write` tool.

## Flags

| Flag               | Default                      | Description                                        |
| ------------------ | ---------------------------- | -------------------------------------------------- |
| `--provider`       | `zai`                        | Model provider to run against, or one named in config |
| `--config`         | `~/.config/acof/config.yaml` | Path to the config file (or `$ACOF_CONFIG`)        |
| `--model`          | `glm-5.2`                    | Model the agent reasons with (overrides config)    |
| `--dir`            | `.`                          | Working directory the agent investigates           |
| `--max-iterations` | `10000`                      | Maximum agent iterations before a forced stop      |
| `--objectives-dir` | `./.acof/objectives`         | Where this project's objectives live, run by a bare `acof` |
| `--records-dir`    | `./.acof/records`            | Where run records (the ledger) are written         |
| `--session-dir`    | `./.acof/sessions`           | Where session logs are written (`--no-session` to disable) |
| `--resume`         | -                            | Continue an earlier session: an id, a path, or `last` |
| `--run-dir`        | `~/.local/state/acof/runs`   | Base directory for per-run artifacts               |
| `--watch`          | `false`                      | Stay up and run objectives as they arrive          |
| `--rerun`          | `false`                      | Run objectives even when the ledger says done      |
| `-v`, `--verbose`  | `false`                      | Stream the agent's reasoning tokens to stdout      |
| `-V`, `--version`  | -                            | Print version and exit                             |

Flags override `ACOF_*` environment variables, which override the config file,
which overrides the built-in defaults.

## Files & directories

Acof uses four distinct locations - it helps to keep them straight:

| Location          | What it holds                                                                                                                                               | Default path                                                                                                     |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Workspace**     | The directory Acof works _in_ - what it reads, edits, and runs commands against. Any file the agent writes (a report you asked for, a PoC) lands here.      | the current working directory (override with `--dir`)                                                            |
| **Dossier**       | The project's objectives and run records: `.acof/objectives/<slug>.yaml` (the mission files) and `.acof/records/<slug>/<run>.yaml` (the ledger receipts).    | `./.acof/` (override with `--objectives-dir` / `--records-dir`)                                                  |
| **Run artifacts** | Acof's own record of _each run_: `status.json` (live state) and `events.jsonl` (append-only log). Telemetry, not work product - the status widget reads it. | `~/.local/state/acof/runs/<runid>/` (`$XDG_STATE_HOME`; override with `--run-dir` / `run_dir` / `$ACOF_RUN_DIR`) |
| **Config**        | Your settings and provider keys.                                                                                                                            | `~/.config/acof/config.yaml` (`$ACOF_CONFIG` / `--config`)                                                       |

The **dossier** is the contract: objective files are what a run is dispatched
from, and the ledger records what has been done. The **run artifacts** are
Acof's log of _what it did_; the **workspace** is _where it did it_. They never
mix: run artifacts are telemetry under your state directory, while the agent's
file writes stay in the workspace.

Each run gets its own `runs/<runid>/` directory (`<timestamp>-<pid>`), so
concurrent runs never overwrite each other; the desktop widget shows the most
recent active run.
