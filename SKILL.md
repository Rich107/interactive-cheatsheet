---
name: interactive-cheatsheet
description: Generate a topic cheatsheet as paired markdown + self-contained interactive HTML. The HTML has a sticky variables panel at the top that live-updates every code snippet, and per-snippet Copy buttons. Use when the user asks for a command reference, runbook, cheatsheet, "commands list", how-to with copy buttons, or any doc where the reader will want to swap in their own values (db name, region, namespace, container, project id, etc.).
---

# Interactive Cheatsheet Skill

Produce two paired artifacts for any command/runbook/how-to topic:

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

### 1. Decide the file location & names

If the user is in a project repo, default to `docs/<topic-slug>.{md,html}` (create `docs/` if missing — most repos already have one). Otherwise ask. The two files should share the same basename so they're obviously paired.

Confirm with the user only if location is ambiguous. Don't ask permission to write — they invoked the skill, that's the permission.

### 2. Identify variables

Pick the values a future reader will most plausibly want to swap. Typical examples by topic:

- **Postgres / SQL** — host, port, admin_user, admin_password, db_name, schema, table, owner
- **Docker / containers** — image, tag, container_name, network, volume, port_host, port_container
- **Kubernetes** — namespace, context, deployment, service, container, image
- **AWS CLI** — profile, region, bucket, function_name, role_arn
- **gcloud** — project, region, zone, service_name
- **git** — branch, remote, base_branch, repo_url
- **Terraform** — workspace, region, var_file
- **systemd** — service, unit_name, user

Aim for **5–10 variables**. Too few makes the panel useless; too many makes it noisy. Give each a sensible default value (e.g. `localhost`, `5432`, `us-east-1`, `my_app`).

Use **snake_case** for variable ids — they become both the input `data-var` and the `{{token}}` inside snippets.

### 3. Write the markdown file

Plain Markdown, no front-matter unless the repo conventionally uses some. Structure:

- `# <Topic title>` heading
- Short intro paragraph (1–2 sentences) on what the page covers
- Numbered `## N. Step name` sections, each containing prose + fenced code blocks (`bash`, `sql`, `yaml`, etc.)
- A final "one-shot" / "TL;DR" combined block when it makes sense

Use the **default values** of the variables literally inside the markdown snippets (e.g. write `my_new_db`, not `{{db_name}}`) — markdown viewers don't do substitution.

If the user has a project-level markdown table preference (e.g. column-aligned pipes), follow it.

### 4. Build the HTML file from the template

Start from `resources/template.html` in this skill folder. Substitute the four template tokens:

| Token              | What to put there                                                                                            |
| ------------------ | ------------------------------------------------------------------------------------------------------------ |
| `{{PAGE_TITLE}}`   | Same title as the markdown `#` heading                                                                       |
| `{{PAGE_SUBTITLE}}`| One-sentence helper line ("Change the variables below — every snippet updates live.")                        |
| `{{VARIABLES}}`    | One `<div class="field">…</div>` per variable (see snippet below)                                            |
| `{{CONTENT}}`      | The body: `<h2>` section headers, `<p>` intros, and `<div class="block">` code blocks (snippet below)        |
| `{{DEFAULTS_JSON}}`| A JS object literal with the same defaults: `{ db_name: "my_new_db", port: "5432", … }`                      |

**Variable input markup:**

```html
<div class="field">
  <label for="db_name">database name</label>
  <input id="db_name" data-var="db_name" value="my_new_db" autocomplete="off" />
</div>
```

**Code block markup (with explain panel):**

```html
<div class="block">
  <span class="lang">bash</span>
  <button class="copy" type="button">Copy</button>
  <pre><code data-template="psql -h {{host}} -p {{port}} -U {{admin_user}}"></code></pre>
  <details class="explain">
    <summary>Explain</summary>
    <div class="explain-body">
      <p>Opens an interactive psql session against the server at
        <code class="tmpl" data-template="{{host}}:{{port}}"></code> as
        <code class="tmpl" data-template="{{admin_user}}"></code>.</p>
      <h4>Flags used</h4>
      <dl class="flags">
        <dt><code>-h, --host=HOSTNAME</code></dt>
        <dd>Database server host or socket directory.</dd>
        <dt><code>-p, --port=PORT</code></dt>
        <dd>Database server port (defaults to 5432).</dd>
        <dt><code>-U, --username=USERNAME</code></dt>
        <dd>Database user name (defaults to your OS user).</dd>
      </dl>
      <h4>Other flags you might want</h4>
      <dl class="flags">
        <dt><code>-d, --dbname=DBNAME</code></dt>
        <dd>Connect to a specific database immediately.</dd>
        <dt><code>-c, --command=COMMAND</code></dt>
        <dd>Run a single SQL command and exit — useful for scripting.</dd>
        <dt><code>-f, --file=FILENAME</code></dt>
        <dd>Execute commands from a file, then exit.</dd>
      </dl>
    </div>
  </details>
</div>
```

**Multi-line snippets** go in a single `data-template` attribute — newlines inside the attribute value are preserved. Indent the inner content however you want; it will render verbatim.

**Escape `<` and `>` inside `data-template`** as `&lt;` and `&gt;` (the runtime decodes them before substitution). This matters for things like heredocs (`&lt;&lt;'SQL'`) and shell redirections.

**Backslashes** (e.g. psql `\c`, `\l`, `\dt`) are fine as-is.

**Template substitution works on any element with `data-template`**, not just `<code>` blocks — so an explanation paragraph can include live values like `<code class="tmpl" data-template="{{db_name}}"></code>`.

### 5. Populate the Explain panel from real `--help` output

Each CLI command in a code block should have an accompanying `<details class="explain">` panel. To author it:

1. **Run `--help`** (or `-h`, or `man <cmd>`) on the host to get authentic flag documentation. Examples: `psql --help`, `createdb --help`, `docker run --help`, `kubectl get --help`, `aws s3 cp help`. These commands are read-only and safe to execute. Capture the output once and reuse across blocks that share the same tool.
2. **Description** — one or two sentences in plain English describing what the *specific* invocation does. Mention the live values via `data-template` spans where it helps.
3. **Flags used** — list every flag that actually appears in this snippet, with the exact short/long forms and the description copied (lightly rewritten if needed) from `--help`.
4. **Other flags you might want** — 3–6 alternative flags from the same `--help` that are commonly useful for related scenarios (different output formats, dry-run modes, force overrides, alternative connection options, etc.). Pick flags that broaden the user's mental model of the tool.

For **SQL statements** or other non-CLI snippets (e.g. `CREATE DATABASE`, `kubectl apply -f`), `--help` isn't available. Instead:

- Describe what the statement does.
- Under "Flags used" list the clauses present (`OWNER`, `ENCODING`, `TEMPLATE`, etc.) with brief descriptions.
- Under "Other clauses you might want" list 3–6 alternatives (`TABLESPACE`, `CONNECTION LIMIT`, `IS_TEMPLATE`, etc.) from the official docs.

If a snippet is trivial (a single `\c db_name`, a `\l`), an Explain panel may be unnecessary — use judgement.

### 6. Default to placing the variables panel before the first section, then keep section order matching the markdown

The HTML should mirror the markdown's structure so the two files feel like the same doc in two skins.

### 7. Offer to open the HTML in the browser

After writing both files, offer (or just do, if the user is clearly happy with the result) `open <path>` to launch it.

## House style

- **Dark theme only** — the template is dark-themed by design. Don't add a light-mode toggle unless asked.
- **No emojis** in the page content (the template itself has none).
- **No external fonts / CDNs** — everything stays inline.
- **One topic per page** — if the user asks for two unrelated topics, make two pages.
- **Don't add explanatory comments** to the generated HTML beyond what the template already contains.

## Verification before finishing

- Every `{{var}}` token inside a `data-template` references a variable that exists in `DEFAULTS` and has an input with the matching `data-var`. Otherwise the token will render as literal `{{name}}`.
- All `<` and `>` characters inside `data-template` attribute values are escaped.
- The markdown file uses literal default values (not `{{tokens}}`).
- The two files share a basename and live in the same directory.
- Every non-trivial code block has a `<details class="explain">` panel with description + flags-used + other-flags, sourced from real `--help` output where available.

## Example invocation

> User: "give me commands for setting up a kafka topic, save it as a skill-style page in docs/"

Produce:

- `docs/kafka-topic-setup.md` — sections: prerequisites, create topic, list topics, describe topic, produce/consume test, delete topic.
- `docs/kafka-topic-setup.html` — same content, with variables `bootstrap_servers`, `topic`, `partitions`, `replication_factor`, `group_id`, `key_serializer`, `value_serializer`.
