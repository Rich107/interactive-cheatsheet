# interactive-cheatsheet

A **topic-agnostic** [Claude Code](https://claude.com/claude-code) skill that turns any "give me a list of commands for X" request into a **paired markdown + interactive HTML cheatsheet**. Works for Postgres, Docker, Kubernetes, AWS CLI, gcloud, git, Terraform, ffmpeg, openssl, curl, systemd, jq, or any other CLI ecosystem.

The HTML page has:

- A sticky **Variables** panel at the top — change a value and every code snippet on the page updates instantly.
- A **Copy** button on every code block that copies the *rendered* command (with your values substituted in).
- An expandable **Explain** panel under each command that describes what it does, lists every flag used (sourced from the tool's real `--help` output), and suggests other useful flags.
- A **Reset to defaults** button.
- Dark theme, fully self-contained — no external CSS, JS, or fonts. Works offline and from `file://`.

## How it works

When the skill triggers, Claude:

1. **Asks which CLI tool(s)** you want the cheatsheet to cover — most topics have competing tools (kubectl vs helm, docker vs podman, npm vs pnpm, etc.) so it confirms first rather than guessing.
2. **Runs each tool's `--help`** to capture real, authoritative flag documentation — falling back to `man`, `tldr`, or docs knowledge when help isn't available.
3. **Writes both files** — a plain `.md` for normal markdown viewers, and the interactive `.html` with the variables panel and per-command Explain panels.

## Examples

> "give me a list of commands to make a new database and schema and table in a postgres database, save it as an interactive cheatsheet"

Claude asks which tools you want (psql + createdb? pg_dump? pgcli?), then produces `docs/postgres-setup.{md,html}` with inputs for `host`, `port`, `admin_user`, `db_name`, `schema`, `table`, `owner` and Explain panels sourced from real `psql --help` / `createdb --help` output.

> "make me a cheatsheet for kubectl pod debugging"

Claude asks whether to include `kubectl exec` / `logs` / `describe` / `port-forward`, then produces `docs/kubectl-pod-debugging.{md,html}` with inputs for `namespace`, `context`, `pod`, `container` and Explain panels from `kubectl exec --help`, `kubectl logs --help`, etc.

> "ffmpeg transcoding commands as an HTML cheatsheet"

Inputs: `input_file`, `output_file`, `video_codec`, `audio_codec`, `bitrate`, `crf`, `start_time`, `duration`. Explain panels from `ffmpeg -h` and `ffmpeg -h encoder=libx264`.

## Ideas — what to ask for

Anything where you'd benefit from a personal, parameterised reference card. A starter menu, grouped by domain:

### Databases & data

- "psql day-to-day cheatsheet — connect, query, dump, restore"
- "redis-cli — keys, expiry, pub/sub, persistence"
- "mongo shell — find, aggregate, indexes, replica sets"
- "sqlite cli — common queries, .schema, .dump, attaching files"
- "duckdb — read parquet/csv, joins, exporting results"

### Containers & orchestration

- "docker daily-driver commands — build, run, exec, logs, prune"
- "docker compose workflows — up/down, scaling, overrides, healthchecks"
- "kubectl pod debugging — exec, logs, describe, port-forward, cp"
- "helm — install, upgrade, rollback, template, values overrides"
- "minikube / kind — cluster lifecycle, registry, ingress"

### Cloud CLIs

- "aws s3 + s3api — copy, sync, presigned URLs, bucket policies"
- "aws ec2 — describe-instances, start/stop, ssh via SSM"
- "aws logs — tail, filter, export to S3"
- "gcloud compute + gcloud sql — VMs, IAM, Cloud SQL connect"
- "az cli — login, subscription switch, resource groups, App Service"

### Version control & GitHub

- "git rescue commands — reflog, restore, cherry-pick, bisect"
- "git branch workflow — rebase, force-push safely, worktrees"
- "gh CLI — PRs, issues, gists, releases, workflow runs"
- "git submodules — add, update, remove, sync"

### Build, package, release

- "npm / pnpm / yarn equivalent commands side-by-side"
- "cargo — new project, test, bench, publish, workspaces"
- "maven / gradle quickrefs"
- "homebrew — taps, casks, formula updates, pinning"
- "release a new version of an npm package end-to-end"

### Networking, security, certificates

- "curl — auth, multipart upload, retries, --resolve, timing"
- "openssl certs — generate, inspect, convert formats, verify chains"
- "ssh + ssh-keygen — keys, agent, config, port forwarding"
- "nmap — host discovery, version detection, scripts"
- "wireguard — server + client config, QR for mobile"

### Media & files

- "ffmpeg — transcoding, trimming, audio extraction, concat"
- "imagemagick — resize, convert format, batch processing"
- "pandoc — md → html / pdf / docx, citations, templates"
- "exiftool — read, write, strip metadata"
- "tar + zstd — fast archives, splitting, encrypting"

### Infrastructure as code & ops

- "terraform — init, plan, apply, state surgery, workspaces"
- "ansible — ad-hoc commands, playbook run, inventory, vault"
- "systemd / systemctl — manage units, journalctl recipes"
- "cron + at — schedule, list, debug"

### Text processing & shell power tools

- "jq — pick, filter, group_by, recursive descent"
- "awk one-liners — sums, counts, field manipulation"
- "ripgrep + fd — common search patterns, replace, exclude"
- "sed — in-place replace, ranges, multi-line"
- "fzf — interactive picker, integration with shell, vim, tmux"

### Local dev environments

- "homebrew — installing/uninstalling, services, doctor"
- "asdf / mise — managing language versions"
- "tmux — sessions, panes, copy mode, scripting"
- "neovim — LSP, telescope, treesitter quickref"
- "direnv — .envrc patterns, layouts, allow/deny"

### Observability

- "kubectl + stern logs workflow"
- "promtool — query, check rules, debug expressions"
- "loki logcli queries"
- "datadog CLI / API recipes"

### Languages / runtimes

- "python venv + pip + uv quickref"
- "node + nvm + corepack version juggling"
- "go modules — init, vendor, replace, build tags"
- "rust toolchain — rustup, cargo, clippy, fmt"

### One-off personal runbooks

- "setting up a new Mac for development from scratch"
- "rotating a leaked API key across our services"
- "publishing a static site to GitHub Pages"
- "onboarding a new engineer to repo X"
- "recovering from a bad force-push on main"

Any of these become a single sentence to Claude — it'll ask which specific CLIs to cover, fetch real `--help` output, and write you both files.

<img width="1598" height="1580" alt="image" src="https://github.com/user-attachments/assets/ff41dbe4-aace-48c9-a931-f222e29ac8d3" />



## When the skill triggers

Claude picks this skill up automatically when you ask for:

- "give me a list of commands to …"
- "make a cheatsheet / runbook / quick reference for …"
- "commands for setting up …"
- An HTML doc with copy buttons and/or live-editable variables
- Saving a previously-generated commands list as a reusable page

You can also force it with `/interactive-cheatsheet` once installed.

## Install

Skills live in `~/.claude/skills/<skill-name>/`. Clone this repo into that location:

```bash
git clone https://github.com/Rich107/interactive-cheatsheet.git ~/.claude/skills/interactive-cheatsheet
```

That's it. Restart Claude Code (or start a new session) and the skill will appear in the available-skills list.

### Update later

```bash
cd ~/.claude/skills/interactive-cheatsheet && git pull
```

### Uninstall

```bash
rm -rf ~/.claude/skills/interactive-cheatsheet
```

## What's in the repo

```
.
├── SKILL.md              # frontmatter + workflow instructions Claude reads
├── resources/
│   └── template.html     # the dark-themed HTML template with {{TOKEN}} substitution points
└── README.md
```

`SKILL.md` is what Claude actually reads when the skill triggers — it explains how to pick variables, how to fill in the template, and the house style (dark theme, no CDNs, snake_case variable ids, etc.).

`resources/template.html` is the boilerplate. It exposes five substitution tokens that Claude fills in per topic:

| Token              | Purpose                                                                       |
| ------------------ | ----------------------------------------------------------------------------- |
| `{{PAGE_TITLE}}`   | Page heading                                                                  |
| `{{PAGE_SUBTITLE}}`| One-line helper text                                                          |
| `{{VARIABLES}}`    | One labelled `<input>` per editable variable                                  |
| `{{CONTENT}}`      | The body — `<h2>` sections, prose, and code blocks                            |
| `{{DEFAULTS_JSON}}`| JS object literal of defaults, used by the Reset button                       |

Inside any code block, `{{var_name}}` references the input with matching `data-var="var_name"` and is replaced live as the user types.

## Customising the template

Edit `resources/template.html` to change colours, font sizes, layout, etc. The defaults are a dark theme tuned for technical reading — feel free to fork.

## License

MIT.
