# chrome-to-memory

A Claude Code skill that reads your Chrome browsing history and writes a dated reading summary to Apple Notes — ready for a notes-wiki ingest.

- **`/chrome-summary`** — filters noise from your history and writes a summary note to a **Browser Summaries** folder in Apple Notes.
- **`/notes-wiki`** — a separate skill (not included here) that ingests notes like these into structured wiki pages under `memory/Wiki`.

## Requirements

- macOS
- Google Chrome (with a browsing history)
- Apple Notes with iCloud sync enabled
- [Claude Code](https://claude.ai/code) CLI

## Installation

**1. Clone or download this repository**

```bash
git clone <repo-url> ~/Documents/chrome-to-memory
```

**2. Grant Full Disk Access to your terminal**

Chrome's history database is protected. Your terminal app needs Full Disk Access to read it.

- Open **System Settings → Privacy & Security → Full Disk Access**
- Click `+` and add your terminal app (Terminal, iTerm2, Warp, etc.)
- Fully quit and relaunch your terminal after granting access

**3. Grant Automation access to Notes**

The skill writes notes via AppleScript, which requires Automation permission.

- Open **System Settings → Privacy & Security → Automation**
- Find your terminal app and toggle **Notes** on

If you see `Not authorized to send Apple events to Notes (-1743)`, this permission is missing.

**4. Open the project in Claude Code**

```bash
cd ~/Documents/chrome-to-memory
claude
```

The `.claude/skills/` folder is picked up automatically. `/chrome-summary` will appear in Claude Code's slash command list.

## Installing notes-wiki (optional)

`/chrome-summary` writes notes to Apple Notes. If you also want those notes turned into a structured wiki, install the [`notes-wiki` skill](https://github.com/RaemondBW/Notes-LLM-Wiki) separately:

```bash
git clone https://github.com/RaemondBW/Notes-LLM-Wiki ~/Documents/notes-wiki
```

Then copy its `SKILL.md` into your project (or global) Claude Code skills directory:

```bash
# Install into this project
mkdir -p .claude/skills/notes-wiki
cp ~/Documents/notes-wiki/SKILL.md .claude/skills/notes-wiki/SKILL.md

# Or install globally (available in every project)
mkdir -p ~/.claude/skills/notes-wiki
cp ~/Documents/notes-wiki/SKILL.md ~/.claude/skills/notes-wiki/SKILL.md
```

Once installed, run `/notes-wiki init` once to set up the Apple Notes folder structure, then `/notes-wiki` (or `/loop /notes-wiki`) to keep the wiki current.

## Usage

**Summarize yesterday's browsing** (default — run once a day to keep up):
```
/chrome-summary
```

**Summarize a specific past day:**
```
/chrome-summary 2026-05-05
```

**Regenerate an existing summary:**
```
/chrome-summary 2026-05-05 --force
```

Summaries are written to an Apple Notes folder called **Browser Summaries**. Each note is titled `Browser Summary — YYYY-MM-DD`.

**Run on a schedule** (e.g. every hour):
```
/loop 60m /chrome-summary
```

## What gets filtered out

The skill strips social feeds, video platforms, search results, email, maps, and shopping — anything that's navigation rather than reading. What comes through: articles, GitHub repos, documentation, HN threads, product pages, and anything with real content.

See the `BLOCKED_DOMAINS` list in `.claude/skills/chrome-summary/SKILL.md` to customize.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `unable to open database file` | Grant Full Disk Access, then fully relaunch your terminal |
| `Not authorized to send Apple events to Notes (-1743)` | Enable Automation → Notes in System Settings |
| Summary is empty | All pages that day were filtered; or Chrome wasn't used |
| Note lands in wrong folder | Run `/chrome-summary` once to auto-create the "Browser Summaries" folder |
