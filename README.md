# interactive-cheatsheet

A [Claude Code](https://claude.com/claude-code) skill that turns any "give me a list of commands for X" request into a **paired markdown + interactive HTML cheatsheet**.

The HTML page has:

- A sticky **Variables** panel at the top — change a value and every code snippet on the page updates instantly.
- A **Copy** button on every code block that copies the *rendered* command (with your values substituted in).
- A **Reset to defaults** button.
- Dark theme, fully self-contained — no external CSS, JS, or fonts. Works offline and from `file://`.

## Example

> "give me a list of commands to make a new database and schema and table in a postgres database, save it as an interactive cheatsheet"

Claude produces two files in your repo's `docs/` folder:

- `docs/postgres-setup.md` — plain markdown reference.
- `docs/postgres-setup.html` — interactive page with inputs for `host`, `port`, `admin_user`, `db_name`, `schema`, `table`, `owner`, etc. Type a new database name → every snippet updates. Click Copy → clipboard has the substituted command.

![cheatsheet preview placeholder — variables panel at top, code blocks with copy buttons below](./preview.png)

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
