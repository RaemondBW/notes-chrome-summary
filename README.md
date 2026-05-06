# chrome-to-memory

Two Claude Code skills that turn your Chrome browsing history into a searchable Apple Notes wiki.

- **`/chrome-summary`** — reads Chrome's history, filters noise, and writes a dated reading summary to Apple Notes.
- **`/notes-wiki`** — ingests notes (including those summaries) into structured wiki pages under a `memory/Wiki` folder.

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

Or just download and unzip it — the only thing that matters is the `.claude/` folder.

**2. Grant Full Disk Access to your terminal**

Chrome's history database and Apple's NoteStore.sqlite are both protected. Your terminal app needs Full Disk Access to read them.

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

The `.claude/skills/` folder is picked up automatically. The skills will appear in Claude Code's slash command list.

## Usage

**Summarize today's browsing:**
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

**Run the wiki ingest** (pulls Browser Summaries and other notes into `memory/Wiki`):
```
/notes-wiki
```

**Run on a schedule** (e.g. every evening):
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
