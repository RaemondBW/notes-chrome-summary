---
name: chrome-summary
description: Summarize today's interesting Chrome browsing into an Apple Notes entry in the "Browser Summaries" folder, ready for notes-wiki to ingest.
argument-hint: '[YYYY-MM-DD] [--force]'
allowed-tools: Bash, Read, Write
user-invocable: true
---

# chrome-summary

Reads Chrome's local history SQLite database, filters out noise (social feeds, video, search results, email), groups what remains into themes, and writes a dated reading summary to Apple Notes in a **"Browser Summaries"** folder outside `memory/`. Because it lands outside `memory/`, the notes-wiki skill picks it up on its next ingest cycle and integrates it into the wiki automatically.

## Arguments

- **No argument** — summarize yesterday's browsing (so running once a day always fills in the previous day)
- **`YYYY-MM-DD`** — summarize a specific past day
- **`--force`** — overwrite the note if one for that date already exists (default: skip if it exists)

Example: `/chrome-summary 2026-05-05 --force`

## Permissions required

- **Full Disk Access** for your terminal app — needed to read Chrome's history DB and Apple's NoteStore.sqlite. Grant in System Settings → Privacy & Security → Full Disk Access.
- **Automation → Notes** — needed to write the summary note via osascript. Grant in System Settings → Privacy & Security → Automation.

Chrome's DB is always copied to a temp file before querying so it doesn't matter if Chrome is open.

---

## Step 1: Parse arguments

Extract `TARGET_DATE` (default `today`) and `FORCE` flag from `ARGUMENTS`.

```bash
# Parse arguments
ARGS="${ARGUMENTS:-}"
FORCE=0
TARGET_DATE=""

for arg in $ARGS; do
  if [[ "$arg" == "--force" ]]; then
    FORCE=1
  elif [[ "$arg" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]; then
    TARGET_DATE="$arg"
  fi
done

if [ -z "$TARGET_DATE" ]; then
  TARGET_DATE=$(date -v-1d +%Y-%m-%d)
fi

echo "target_date=$TARGET_DATE force=$FORCE"
```

---

## Step 2: Preflight checks

```bash
CHROME_DB="$HOME/Library/Application Support/Google/Chrome/Default/History"
NOTES_DB="$HOME/Library/Group Containers/group.com.apple.notes/NoteStore.sqlite"

# Chrome history readable?
if [ ! -f "$CHROME_DB" ]; then
  echo "ERROR: Chrome history not found at $CHROME_DB"
  exit 1
fi
cp "$CHROME_DB" /tmp/chrome_summary_$$.db

# Apple Notes SQLite readable?
if sqlite3 "$NOTES_DB" "SELECT 1;" >/dev/null 2>&1; then SQLITE_OK=1; else SQLITE_OK=0; fi

# Apple Notes Automation?
if osascript -e 'tell application "Notes" to get name of every account' >/dev/null 2>&1; then
  AUTOMATION_OK=1
else
  AUTOMATION_OK=0
fi

echo "preflight: chrome=1 notes_sqlite=$SQLITE_OK automation=$AUTOMATION_OK"

if [ "$AUTOMATION_OK" -eq 0 ]; then
  echo "ERROR: Automation permission for Notes is denied."
  echo "Open System Settings → Privacy & Security → Automation, find your terminal, and toggle Notes on."
  rm -f /tmp/chrome_summary_$$.db
  exit 1
fi
```

---

## Step 3: Check if note for this date already exists

Use NoteStore.sqlite (fast path) or AppleScript fallback to check whether a note titled `Browser Summary — YYYY-MM-DD` already exists in the "Browser Summaries" folder.

```bash
TARGET_DATE="$TARGET_DATE"  # set in Step 1
NOTE_TITLE="Browser Summary — $TARGET_DATE"
NOTES_DB="$HOME/Library/Group Containers/group.com.apple.notes/NoteStore.sqlite"
TITLE_SQL=$(printf '%s' "$NOTE_TITLE" | sed "s/'/''/g")

if [ "$SQLITE_OK" -eq 1 ]; then
  EXISTING_ID=$(sqlite3 "$NOTES_DB" "
    SELECT 'x-coredata://' || (SELECT Z_UUID FROM Z_METADATA LIMIT 1) || '/ICNote/p' || obj.Z_PK
    FROM ZICCLOUDSYNCINGOBJECT obj
    JOIN ZICCLOUDSYNCINGOBJECT folder ON obj.ZFOLDER = folder.Z_PK
    WHERE folder.ZTITLE2 = 'Browser Summaries'
      AND obj.ZTITLE1 = '$TITLE_SQL'
      AND obj.ZMARKEDFORDELETION = 0
    LIMIT 1;
  ")
else
  EXISTING_ID=$(osascript <<EOF
tell application "Notes"
  tell account "iCloud"
    repeat with n in notes
      try
        if container of n is not missing value then
          if name of (container of n) is "Browser Summaries" then
            if name of n is "$NOTE_TITLE" then return id of n
          end if
        end if
      end try
    end repeat
  end tell
  return ""
end tell
EOF
)
fi

if [ -n "$EXISTING_ID" ] && [ "$FORCE" -eq 0 ]; then
  echo "Note for $TARGET_DATE already exists (id: $EXISTING_ID). Use --force to overwrite."
  rm -f /tmp/chrome_summary_$$.db
  exit 0
fi
echo "existing_id=${EXISTING_ID:-none}"
```

---

## Step 4: Ensure "Browser Summaries" folder exists in Apple Notes

Run once per invocation (idempotent).

```bash
osascript <<'EOF'
tell application "Notes"
  tell account "iCloud"
    if not (exists folder "Browser Summaries") then
      make new folder with properties {name:"Browser Summaries"}
    end if
  end tell
end tell
return "folder ready"
EOF
```

---

## Step 5: Query Chrome history for the target date

Chrome timestamps are microseconds since 1601-01-01 UTC. Subtract `11644473600` seconds and divide by `1000000` to get Unix seconds.

### Blocklist — domains to filter out

These patterns are passed to a Python filter. Add more as needed.

```python
BLOCKED_DOMAINS = {
    # Social media feeds (homepages / feeds only; specific posts may still be interesting)
    "twitter.com", "x.com", "facebook.com", "instagram.com",
    "linkedin.com", "tiktok.com", "snapchat.com", "threads.net",
    "bsky.app", "mastodon.social", "pinterest.com", "reddit.com",

    # Video platforms
    "youtube.com", "youtu.be", "vimeo.com", "twitch.tv",
    "netflix.com", "hulu.com", "disneyplus.com", "primevideo.com",
    "peacocktv.com", "dailymotion.com", "rumble.com",

    # Search engines
    "google.com", "bing.com", "duckduckgo.com", "search.yahoo.com",
    "ecosia.org",

    # Email / Calendar / Maps / Drive
    "mail.google.com", "calendar.google.com", "drive.google.com",
    "docs.google.com", "sheets.google.com", "slides.google.com",
    "outlook.live.com", "outlook.office.com", "maps.google.com",

    # Shopping
    "amazon.com", "ebay.com", "etsy.com", "walmart.com", "bestbuy.com",
    "target.com", "wayfair.com",

    # Package tracking / banking (noise, not reading)
    "fedex.com", "ups.com", "usps.com",

    # Chrome internals
    "chrome", "about",
}

# URL path prefixes that are pure navigation, not content
BLOCKED_PATH_PREFIXES = [
    "/search?", "/search/?", "?q=", "?query=", "?s=",
]

# Pages that are just bare domain homepages (path is "/" or empty) are often not reading
# We keep them if they're on an always-interesting list (e.g. news.ycombinator.com)
ALWAYS_INTERESTING_DOMAINS = {
    "news.ycombinator.com",
    "lobste.rs",
    "tildes.net",
    "slashdot.org",
}
```

### Shell: Extract and filter URLs

```bash
TARGET_DATE="$TARGET_DATE"

# Chrome epoch offset: 11644473600 seconds
python3 - /tmp/chrome_summary_$$.db "$TARGET_DATE" <<'PYEOF'
import sys, sqlite3, urllib.parse

db_path, target_date = sys.argv[1], sys.argv[2]

BLOCKED_DOMAINS = {
    "twitter.com","x.com","facebook.com","instagram.com","linkedin.com",
    "tiktok.com","snapchat.com","threads.net","bsky.app","mastodon.social",
    "pinterest.com","reddit.com","youtube.com","youtu.be","vimeo.com","twitch.tv",
    "netflix.com","hulu.com","disneyplus.com","primevideo.com","peacocktv.com",
    "dailymotion.com","rumble.com","google.com","bing.com","duckduckgo.com",
    "search.yahoo.com","ecosia.org","mail.google.com","calendar.google.com",
    "drive.google.com","docs.google.com","sheets.google.com","slides.google.com",
    "outlook.live.com","outlook.office.com","maps.google.com","amazon.com",
    "ebay.com","etsy.com","walmart.com","bestbuy.com","target.com","wayfair.com",
    "fedex.com","ups.com","usps.com",
}
BLOCKED_SCHEMES = {"chrome", "about", "data", "javascript", "blob"}
ALWAYS_INTERESTING = {"news.ycombinator.com","lobste.rs","tildes.net","slashdot.org"}

EPOCH_OFFSET = 11644473600  # seconds between 1601-01-01 and 1970-01-01

con = sqlite3.connect(db_path)
rows = con.execute("""
    SELECT DISTINCT u.url, u.title,
           MIN(v.visit_time) as first_visit,
           MAX(v.visit_time) as last_visit,
           COUNT(*) as visit_count
    FROM visits v
    JOIN urls u ON v.url = u.id
    WHERE date((v.visit_time / 1000000) - ?, 'unixepoch', 'localtime') = ?
    GROUP BY u.url
    ORDER BY first_visit ASC
""", (EPOCH_OFFSET, target_date)).fetchall()
con.close()

seen_urls = set()
results = []
for url, title, first_ts, last_ts, count in rows:
    try:
        parsed = urllib.parse.urlparse(url)
    except Exception:
        continue

    scheme = parsed.scheme.lower()
    if scheme in BLOCKED_SCHEMES:
        continue

    hostname = parsed.netloc.lower().lstrip("www.")
    # Check if any blocked domain is a suffix of the hostname
    is_blocked = any(
        hostname == bd or hostname.endswith("." + bd)
        for bd in BLOCKED_DOMAINS
    )
    if is_blocked:
        continue

    # Filter bare homepages unless always-interesting
    path = parsed.path.rstrip("/")
    is_homepage = (not path or path == "") and not parsed.query
    if is_homepage and hostname not in ALWAYS_INTERESTING:
        continue

    # Filter obvious search/query pages
    query_str = parsed.query.lower()
    if any(p in query_str for p in ["q=", "query=", "search="]):
        # Keep if domain is always interesting even with query params
        if hostname not in ALWAYS_INTERESTING:
            continue

    if url in seen_urls:
        continue
    seen_urls.add(url)

    visit_time_unix = int(first_ts / 1000000) - EPOCH_OFFSET
    results.append((url, title or "(no title)", visit_time_unix, hostname))

for url, title, ts, host in results:
    # Tab-separated: url, title, unix_timestamp, hostname
    print(f"{url}\t{title}\t{ts}\t{host}")

PYEOF
```

Save the output to a variable or temp file for the summarization step.

---

## Step 6: Summarize with LLM

After filtering, you have a list of `url, title, timestamp, hostname` lines. Now synthesize.

**If the list is empty**: print `"No interesting pages found for $TARGET_DATE."` and write a brief note saying so, then exit cleanly.

**If there are results**: group them by theme, not just by domain. Look at titles together and identify what the reader was exploring. Produce a summary in this format:

```
## What I read — {TARGET_DATE}

{2-4 sentence narrative about the day's reading themes. What were the main intellectual threads? What was the reader working on or curious about?}

### Highlights
{For each standout page — a concise one-line note on what it's about. Lead with the most interesting. Max ~10 highlights total.}
- **[Hostname | Title]** — one sentence on why it's notable
- ...

### All interesting pages
{A compact list of every URL that made it through the filter, with title. Group by rough category if there are >8 pages.}

**[Category if applicable]**
- Title — hostname

...
```

The narrative is the most important part. Don't just list — synthesize.

---

## Step 7: Write to Apple Notes

### HTML body preparation

Convert the markdown summary to Apple Notes HTML. Use `<div>` for each line, `<b>` for bold, `<br>` between sections.

```bash
NOTE_TITLE="Browser Summary — $TARGET_DATE"

# Write summary content to a temp file as HTML
BODY_FILE=$(mktemp -t chrome-summary)

python3 - "$BODY_FILE" <<'PYEOF'
import sys, html as html_lib

body_file = sys.argv[1]

# The summary string — replace this with the actual LLM output
# passed via env var or heredoc in practice
summary = """SUMMARY_PLACEHOLDER"""

lines = summary.split("\n")
parts = []
for line in lines:
    if line.strip() == "":
        parts.append("<div><br></div>")
    else:
        # Convert markdown bold to <b>
        import re
        line = re.sub(r'\*\*(.+?)\*\*', r'<b>\1</b>', html_lib.escape(line))
        # Convert markdown headers to bold divs
        if line.startswith("## "):
            parts.append(f"<div><b>{line[3:]}</b></div>")
        elif line.startswith("### "):
            parts.append(f"<div><b>{line[4:]}</b></div>")
        elif line.startswith("- "):
            parts.append(f"<div>• {line[2:]}</div>")
        else:
            parts.append(f"<div>{line}</div>")

with open(body_file, "w") as f:
    f.write("".join(parts))
PYEOF
```

In practice, build the full HTML directly in Python from the filtered URLs and your synthesis — don't use a placeholder. Write the final HTML to `$BODY_FILE`, then:

### Create or update the note

**If `EXISTING_ID` is empty** (new note):

```bash
NOTE_TITLE_AS=$(printf '%s' "$NOTE_TITLE" | sed 's/\\/\\\\/g; s/"/\\"/g')
osascript <<EOF
set bodyText to (read POSIX file "$BODY_FILE" as «class utf8»)
tell application "Notes"
  tell account "iCloud"
    make new note at folder "Browser Summaries" with properties {name:"$NOTE_TITLE_AS", body:bodyText}
  end tell
end tell
EOF
```

**If `EXISTING_ID` is non-empty** (update / `--force`):

```bash
osascript <<EOF
set bodyText to (read POSIX file "$BODY_FILE" as «class utf8»)
tell application "Notes"
  set body of note id "$EXISTING_ID" to bodyText
end tell
EOF
```

Cleanup:
```bash
rm -f "$BODY_FILE" /tmp/chrome_summary_$$.db
echo "Written: $NOTE_TITLE"
```

---

## Step 8: Print result

```
chrome-summary: wrote "Browser Summary — YYYY-MM-DD" to Apple Notes / Browser Summaries
  Pages summarized: N (of M visited today)
  Filtered out: X social/video/search pages
```

The note will be picked up by `/notes-wiki` on its next ingest cycle.

---

## Execution model

Run the full skill **inline** (not as a sub-agent). The URL list is small and the LLM synthesis is the main work — there is no context-accumulation problem here. Steps 5–7 are done in one pass:

1. Run the Step 5 shell block to get the filtered URL list.
2. Read the list and synthesize in your current context (Step 6).
3. Build the HTML and call osascript (Step 7).

---

## Common errors

| Error | Cause | Fix |
|---|---|---|
| `unable to open database file` (Chrome) | File not found or FDA not granted | Check path; grant FDA in System Settings → Privacy & Security → Full Disk Access |
| `unable to open database file` (Notes) | FDA not granted for NoteStore.sqlite | Same FDA grant as above; fully quit and relaunch terminal |
| `Not authorized to send Apple events to Notes (-1743)` | Automation permission denied | System Settings → Privacy & Security → Automation → toggle Notes on for your terminal |
| `Can't make "x-coredata://…" into type integer` | Old AppleScript ID mismatch | Delete stale reference; the note ID changes if Notes rebuilds its database |
| No output from Step 5 | All pages today were filtered, or Chrome not used | Check `BLOCKED_DOMAINS` list; or just nothing to show today |
| Note created in wrong folder | "Browser Summaries" folder name mismatch | Run Step 4 to ensure folder exists; check exact name |

## Tips

- **Reddit**: `reddit.com` is blocked. To allow individual posts/comments back in, remove it from `BLOCKED_DOMAINS` and add a path-based keep-list that requires `/comments/` in the URL.
- **GitHub**: Not blocked; repos, issues, PRs, and discussions come through naturally.
- **Substack / Medium**: Not blocked; article pages come through. Substack homepage (`substack.com`) is a homepage and gets filtered; individual posts (e.g. `authorname.substack.com/p/article`) pass through.
- **Re-running**: Use `--force` to regenerate a note after re-reading things or if the first run was sparse.
- **notes-wiki integration**: The "Browser Summaries" folder is outside `memory/`, so every note in it is a source for notes-wiki. Run `/notes-wiki ingest` (or wait for the `/loop`) to pull the day's reading into the wiki.
