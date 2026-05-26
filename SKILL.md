---
name: interactive-cheatsheet
description: Generate a topic-agnostic cheatsheet for any tool, runbook, or how-to as paired markdown + self-contained interactive HTML. The HTML has a sticky variables panel at the top that live-updates every code snippet, per-snippet Copy buttons, and an Explain panel under each command sourced from the tool's real --help output. Use when the user asks for a command reference, runbook, cheatsheet, "commands list", how-to with copy buttons, or any doc where the reader will want to swap in their own values (db name, region, namespace, container, project id, etc.). Works for any CLI ecosystem — Postgres, Docker, Kubernetes, AWS, gcloud, git, Terraform, ffmpeg, systemd, openssl, curl, anything.
---

# Interactive Cheatsheet Skill

A **topic-agnostic** skill. The same workflow works for Postgres, Docker, Kubernetes, AWS CLI, gcloud, git, Terraform, ffmpeg, openssl, curl, systemd, npm, pnpm, just, jq, awk, or any other CLI / runbook topic. There is nothing Postgres-specific in the template or the workflow — Postgres just happens to be a common example.

Produce two paired artifacts:

1. A plain-markdown reference (`.md`)
2. A self-contained dark-themed HTML file (`.html`) with:
   - A sticky **Variables** panel at the top (text inputs for the values the reader is likely to change)
   - Code blocks that **live-update** as the inputs change (via `{{var_name}}` placeholders inside `data-template` attributes)
   - A **Copy** button on every code block that copies the *rendered* (substituted) text
   - An expandable **Explain** panel under each code block that describes the command, lists every flag used (sourced from the tool's real `--help` output where possible), and suggests other useful flags
   - A **Reset to defaults** button
   - No external CSS / JS / font dependencies — everything inline so it works offline and from `file://`

## When to use

Use this skill when the user asks for:

- "give me a list of commands to …"
- "make a cheatsheet / runbook / quick reference for …"
- "commands for setting up …" / "how do I … (write it down)"
- An HTML doc with copy buttons and/or live-editable variables
- Saving a previously-generated commands list as a reusable page

Skip it for one-off conversational answers where the user clearly just wants the answer in chat.

## Workflow

### Step 1 — Confirm scope with the user (AskUserQuestion)

**Before writing anything**, use the `AskUserQuestion` tool to confirm the scope. The single most important thing to nail down is **which CLI tools** the cheatsheet should cover, because most topics have multiple competing tools and you don't want to write the wrong one.

Ask **one** multi-select question listing the candidate tools, with the most likely default(s) marked as "(Recommended)". Examples of how to frame it by topic:

| User's request                    | Candidate tools to offer                                                          |
| --------------------------------- | --------------------------------------------------------------------------------- |
| "postgres database setup"         | `psql` + `createdb` (Recommended), `pg_dump` / `pg_restore`, `pgcli`              |
| "kubernetes deployment"           | `kubectl` (Recommended), `helm`, `kustomize`, `k9s`                               |
| "docker container basics"         | `docker` CLI (Recommended), `docker compose`, `podman`                            |
| "aws s3 file operations"          | `aws s3` high-level (Recommended), `aws s3api` low-level, `s5cmd`                 |
| "video conversion"                | `ffmpeg` (Recommended), `HandBrakeCLI`, `youtube-dl` / `yt-dlp`                   |
| "ssl certificate management"      | `openssl` (Recommended), `certbot`, `step-cli`, `mkcert`                          |
| "git branch workflow"             | `git` (Recommended), `gh` (GitHub CLI), `git-flow`                                |
| "node.js project setup"           | `npm`, `pnpm`, `yarn`, `bun` (offer the one matching the user's repo if obvious)  |

If the request is unambiguous (e.g. "give me ffmpeg commands for transcoding") you can skip the question, but **default to asking**. A 5-second multi-select beats a 5-minute rewrite.

If you also genuinely don't know where the file should be saved (no project repo context, no `docs/` folder), tack a save-location question on as a second question in the same `AskUserQuestion` call. Otherwise let location default per step 4 below.

### Step 2 — Capture real help output for every tool in scope

For each CLI tool the user confirmed, run its help command and keep the output. **These are read-only commands and safe to execute.** Try in this order until you get usable text:

1. `<cmd> --help`
2. `<cmd> -h`
3. `<cmd> help`  (sub-command-style tools: `git help`, `aws help`, `gh help`)
4. `<cmd> help <subcommand>` for relevant subcommands (e.g. `docker help run`, `kubectl get --help`, `aws s3 cp help`, `git help commit`)
5. `man <cmd>` (last resort — verbose)
6. `tldr <cmd>` (if installed — concise usage examples)

Capture the help for **every command and subcommand** that will appear in the cheatsheet. A `docker compose up` cheatsheet needs `docker compose --help` and `docker compose up --help`, not just `docker --help`.

If a tool isn't installed locally:
- Note it explicitly to the user.
- Fall back to widely-known flags from training knowledge, but mark the Explain content with a note like "Flag descriptions from docs — `<cmd>` not installed locally."
- For SQL / config-language snippets that have no `--help` at all, use documented clauses (see step 7 below).

### Step 3 — Decide the file location & names

If the user is in a project repo, default to `docs/<topic-slug>.{md,html}` (create `docs/` if missing — most repos already have one). If there is no project repo, default to `~/cheatsheets/<topic-slug>.{md,html}` and mention it. The two files must share the same basename so they're obviously paired.

Don't ask permission to write — they invoked the skill, that's the permission.

### Step 4 — Identify variables

Pick the values a future reader will most plausibly want to swap. Build the variable list **from the captured help output and the commands you're about to write** — every flag value the user is likely to customise becomes a variable. Examples by topic:

| Topic               | Likely variables                                                                                |
| ------------------- | ----------------------------------------------------------------------------------------------- |
| Postgres / SQL      | `host`, `port`, `admin_user`, `admin_password`, `db_name`, `schema`, `table`, `owner`           |
| Docker              | `image`, `tag`, `container_name`, `network`, `volume`, `port_host`, `port_container`            |
| Kubernetes          | `namespace`, `context`, `deployment`, `service`, `container`, `image`                           |
| AWS CLI             | `profile`, `region`, `bucket`, `prefix`, `function_name`, `role_arn`                            |
| gcloud              | `project`, `region`, `zone`, `service_name`                                                     |
| git                 | `branch`, `remote`, `base_branch`, `repo_url`                                                   |
| Terraform           | `workspace`, `region`, `var_file`                                                               |
| ffmpeg              | `input_file`, `output_file`, `codec`, `bitrate`, `start_time`, `duration`                       |
| systemd             | `service`, `unit_name`, `user`                                                                  |
| openssl             | `cert_file`, `key_file`, `csr_file`, `subject`, `days`                                          |
| curl                | `url`, `auth_header`, `method`, `output_file`                                                   |

Aim for **5–10 variables**. Too few makes the panel useless; too many makes it noisy. Give each a sensible default value (e.g. `localhost`, `5432`, `us-east-1`, `my_app`, `input.mp4`).

Use **snake_case** for variable ids — they become both the input `data-var` and the `{{token}}` inside snippets.

### Step 5 — Write the markdown file

Plain Markdown, no front-matter unless the repo conventionally uses some. Structure:

- `# <Topic title>` heading
- Short intro paragraph (1–2 sentences) on what the page covers
- Numbered `## N. Step name` sections, each containing prose + fenced code blocks (`bash`, `sql`, `yaml`, etc.)
- A final "one-shot" / "TL;DR" combined block when it makes sense

Use the **default values** of the variables literally inside the markdown snippets (e.g. write `my_new_db`, not `{{db_name}}`) — markdown viewers don't do substitution.

If the user has a project-level markdown table preference (e.g. column-aligned pipes), follow it.

### Step 6 — Build the HTML file from the template

Start from `resources/template.html` in this skill folder. Substitute the five template tokens:

| Token              | What to put there                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------ |
| `{{PAGE_TITLE}}`   | Same title as the markdown `#` heading                                                                       |
| `{{PAGE_SUBTITLE}}`| One-sentence helper line ("Change the variables below — every snippet updates live.")                        |
| `{{VARIABLES}}`    | One `<div class="field">…</div>` per variable                                                                |
| `{{CONTENT}}`      | The body: `<h2>` section headers, `<p>` intros, and `<div class="block">` code blocks                        |
| `{{DEFAULTS_JSON}}`| A JS object literal with the same defaults                                                                   |

**Variable input markup:**

```html
<div class="field">
  <label for="input_file">input file</label>
  <input id="input_file" data-var="input_file" value="input.mp4" autocomplete="off" />
</div>
```

**Code block markup (with explain panel — neutral ffmpeg example):**

```html
<div class="block">
  <span class="lang">bash</span>
  <button class="copy" type="button">Copy</button>
  <pre><code data-template="ffmpeg -i {{input_file}} -c:v libx264 -b:v {{bitrate}} {{output_file}}"></code></pre>
  <details class="explain">
    <summary>Explain</summary>
    <div class="explain-body">
      <p>Re-encodes <code class="tmpl" data-template="{{input_file}}"></code> to H.264 at
        <code class="tmpl" data-template="{{bitrate}}"></code>, writing
        <code class="tmpl" data-template="{{output_file}}"></code>.</p>
      <h4>Flags used</h4>
      <dl class="flags">
        <dt><code>-i FILE</code></dt><dd>Input file. Can be repeated for multiple inputs.</dd>
        <dt><code>-c:v CODEC</code></dt><dd>Video codec (e.g. <code>libx264</code>, <code>libx265</code>, <code>copy</code>).</dd>
        <dt><code>-b:v BITRATE</code></dt><dd>Target video bitrate (e.g. <code>2M</code>, <code>800k</code>).</dd>
      </dl>
      <h4>Other flags you might want</h4>
      <dl class="flags">
        <dt><code>-c:a CODEC</code></dt><dd>Audio codec — pair with <code>-c:v</code> when re-encoding both tracks.</dd>
        <dt><code>-crf N</code></dt><dd>Constant Rate Factor for x264/x265 — quality target instead of bitrate.</dd>
        <dt><code>-ss TIME</code></dt><dd>Seek to a start position (before <code>-i</code> for fast seek).</dd>
        <dt><code>-t DURATION</code></dt><dd>Limit output duration.</dd>
        <dt><code>-y</code></dt><dd>Overwrite the output file without prompting.</dd>
      </dl>
    </div>
  </details>
</div>
```

**Multi-line snippets** go in a single `data-template` attribute — newlines inside the attribute value are preserved. Indent the inner content however you want; it will render verbatim.

**Escape `<` and `>` inside `data-template`** as `&lt;` and `&gt;` (the runtime decodes them before substitution). This matters for heredocs (`&lt;&lt;'EOF'`), shell redirections, XML, JSX, etc.

**Backslashes** (e.g. psql `\c`, `\l`, `\dt`; or shell line-continuations) are fine as-is.

**Template substitution works on any element with `data-template`**, not just `<code>` blocks — so an explanation paragraph can include live values like `<code class="tmpl" data-template="{{db_name}}"></code>`.

### Step 7 — Populate each Explain panel from the captured help output

For each CLI command in a code block, author its `<details class="explain">` panel:

1. **Description** — one or two sentences in plain English describing what the *specific* invocation does. Mention the live values via `data-template` spans where it helps.
2. **Flags used** — list every flag that actually appears in this snippet, with the exact short/long forms and the description copied (lightly rewritten where the help text is unclear) from the captured `--help` output.
3. **Other flags you might want** — 3–6 alternative flags from the same help output that are commonly useful for related scenarios (different output formats, dry-run modes, force overrides, alternative connection options, etc.). Pick flags that broaden the user's mental model of the tool.

For **SQL / config-language / domain-specific snippets** (e.g. `CREATE DATABASE`, `kubectl apply -f`, Dockerfile lines, `crontab` entries, NGINX directives) there is no `--help`. Instead:

- Describe what the statement does.
- Under "Clauses used" / "Directives used" list the parts present with brief descriptions.
- Under "Other clauses you might want" list 3–6 alternatives from the official reference docs.

If a snippet is trivial (a single `\c db_name`, a `\l`, a one-flag command obvious from its name) an Explain panel may be unnecessary — use judgement.

### Step 8 — Verify, then open in the browser

Run through the verification checklist below, then `open <path-to-html>` to launch it.

## Help-discovery cheatsheet (for Claude, not the user)

Different tools surface help differently. Quick reference:

| Tool family                       | How to get help                                                          |
| --------------------------------- | ------------------------------------------------------------------------ |
| GNU / POSIX                       | `cmd --help`, fallback `man cmd`                                         |
| BSD / macOS built-ins             | `man cmd` (often no `--help`)                                            |
| Sub-command CLIs                  | `cmd help`, `cmd help <subcmd>`, `cmd <subcmd> --help`                   |
| git                               | `git help <subcmd>` (opens man); `git <subcmd> -h` for short form        |
| docker / podman                   | `docker help`, `docker <cmd> --help`, `docker compose <cmd> --help`      |
| kubectl                           | `kubectl <verb> --help`, `kubectl explain <resource>`                    |
| aws                               | `aws help`, `aws <service> help`, `aws <service> <op> help`              |
| gcloud                            | `gcloud help`, `gcloud <group> --help`, `gcloud <group> <cmd> --help`    |
| psql, createdb, pg_dump           | `<cmd> --help` (verbose, well-categorised)                               |
| ffmpeg                            | `ffmpeg -h`, `ffmpeg -h full`, `ffmpeg -h encoder=libx264`               |
| openssl                           | `openssl help`, `openssl <cmd> -help` (note: single dash)                |
| curl                              | `curl --help all`, `curl --manual`                                       |
| systemd                           | `systemctl --help`, `man systemd.service`                                |

Always prefer the most specific help (`docker run --help`, not `docker --help`) so you get flag descriptions, not just a list of subcommands.

## House style

- **Dark theme only** — the template is dark-themed by design. Don't add a light-mode toggle unless asked.
- **No emojis** in the page content (the template itself has none).
- **No external fonts / CDNs** — everything stays inline.
- **One topic per page** — if the user asks for two unrelated topics, make two pages.
- **Don't add explanatory comments** to the generated HTML beyond what the template already contains.
- **Don't hard-code anything Postgres-flavoured into a non-Postgres cheatsheet.** Drop variables, language tags, and example flags that aren't relevant to the chosen topic.

## Verification before finishing

- Variable list and Explain content match the **tools the user actually selected** in step 1.
- Every `{{var}}` token inside a `data-template` references a variable that exists in `DEFAULTS` and has an input with the matching `data-var`. Otherwise the token renders as literal `{{name}}`.
- All `<` and `>` characters inside `data-template` attribute values are escaped.
- The markdown file uses literal default values (not `{{tokens}}`).
- The two files share a basename and live in the same directory.
- Every non-trivial code block has a `<details class="explain">` panel with description + flags-used + other-flags, sourced from real captured help output (or marked as docs-derived if the tool wasn't installed).

## Example invocations

> User: "give me commands for setting up a kafka topic, save it as a skill-style page in docs/"

Step 1 question: "Which Kafka CLIs should the cheatsheet cover? (a) `kafka-topics.sh` + `kafka-console-producer/consumer.sh` (Recommended); (b) `confluent` CLI; (c) `kcat`/`kafkacat`." User picks (a).

Step 2: capture `kafka-topics.sh --help`, `kafka-console-producer.sh --help`, `kafka-console-consumer.sh --help`.

Produce `docs/kafka-topic-setup.{md,html}` with variables `bootstrap_servers`, `topic`, `partitions`, `replication_factor`, `group_id`, and Explain panels sourced from the captured help.

---

> User: "make me a cheatsheet for ffmpeg transcoding"

Step 1 question: "Which ffmpeg use cases should the cheatsheet cover (multi-select)? Transcoding (Recommended); Trimming/clipping; Audio extraction; Concatenation; Streaming."

Step 2: capture `ffmpeg -h` and `ffmpeg -h encoder=libx264` etc. for the picked use cases.

Produce paired md+html with variables `input_file`, `output_file`, `video_codec`, `audio_codec`, `bitrate`, `crf`, `start_time`, `duration`.
