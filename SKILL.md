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

**Code block markup:**

```html
<div class="block">
  <span class="lang">sql</span>
  <button class="copy" type="button">Copy</button>
  <pre><code data-template="CREATE DATABASE {{db_name}};"></code></pre>
</div>
```

**Multi-line snippets** go in a single `data-template` attribute — newlines inside the attribute value are preserved. Indent the inner content however you want; it will render verbatim.

**Escape `<` and `>` inside `data-template`** as `&lt;` and `&gt;` (the runtime decodes them before substitution). This matters for things like heredocs (`&lt;&lt;'SQL'`) and shell redirections.

**Backslashes** (e.g. psql `\c`, `\l`, `\dt`) are fine as-is.

### 5. Default to placing the variables panel before the first section, then keep section order matching the markdown

The HTML should mirror the markdown's structure so the two files feel like the same doc in two skins.

### 6. Offer to open the HTML in the browser

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

## Example invocation

> User: "give me commands for setting up a kafka topic, save it as a skill-style page in docs/"

Produce:

- `docs/kafka-topic-setup.md` — sections: prerequisites, create topic, list topics, describe topic, produce/consume test, delete topic.
- `docs/kafka-topic-setup.html` — same content, with variables `bootstrap_servers`, `topic`, `partitions`, `replication_factor`, `group_id`, `key_serializer`, `value_serializer`.
