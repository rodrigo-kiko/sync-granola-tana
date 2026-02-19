# sync-granola-tana

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) Skill that syncs your [Granola](https://granola.ai) meeting notes to [Tana](https://tana.inc) Daily Pages — automatically, with smart deduplication and structured formatting.

![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-blueviolet) ![Granola](https://img.shields.io/badge/Granola-MCP-orange) ![Tana](https://img.shields.io/badge/Tana-MCP-blue)

## What It Does

This skill bridges two powerful tools:

- **Granola** records and summarizes your meetings with AI
- **Tana** organizes your daily work in structured Daily Pages

When you run `/sync-granola-tana`, the skill:

1. Fetches your meetings from the last 7 days via Granola MCP
2. Checks your Tana Daily Pages for existing entries
3. Detects duplicates using Granola transcript links (most reliable) or title/participant matching
4. Shows you a sync plan and asks for confirmation
5. Creates beautifully formatted meeting entries in Tana with summaries, attendees, and transcript links
6. Reports what was created, updated, or skipped

## Example Output in Tana

After syncing, each meeting appears in the correct Daily Page:

```
08:00 1:1 with John #meeting
  ├── Date: Wed, Feb 19
  ├── Attendees: John Doe, Jane Smith
  └── Summary
      ├── Opening / Check-in
      │   ├── Discussed recent project updates
      │   └── Reviewed quarterly goals
      ├── Key Decisions
      │   ├── Move forward with proposal A
      │   └── Schedule follow-up for next week
      ├── Action Items
      │   ├── John to prepare budget estimate
      │   └── Jane to draft project timeline
      └── Chat with meeting transcript: https://notes.granola.ai/t/...
```

## Prerequisites

You need both MCP servers running:

### 1. Granola MCP

Granola's MCP server ships with the Granola desktop app. Make sure:
- Granola desktop app is installed and running
- MCP is enabled in Granola settings
- The Granola MCP server is configured in your Claude Code MCP settings

Add to your `~/.claude/settings.json` or project `.claude/settings.json`:

```json
{
  "mcpServers": {
    "granola": {
      "command": "npx",
      "args": ["-y", "granola-mcp-server"]
    }
  }
}
```

> Check [Granola's documentation](https://granola.ai) for the latest MCP setup instructions.

### 2. Tana MCP

Tana's MCP server runs locally on port 8262. To enable it:

1. Open Tana desktop app
2. Go to **Settings** → **MCP**
3. Enable the MCP server
4. Note the authorization token displayed

The Tana MCP server will be available at `http://127.0.0.1:8262/mcp`.

Add to your Claude Code MCP settings:

```json
{
  "mcpServers": {
    "tana-local": {
      "type": "sse",
      "url": "http://127.0.0.1:8262/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_TANA_MCP_TOKEN"
      }
    }
  }
}
```

> Replace `YOUR_TANA_MCP_TOKEN` with the token from Tana's MCP settings.

## Installation

### Option 1: Clone into Claude Code Skills directory (recommended)

```bash
# Clone the repo into your global skills directory
git clone https://github.com/rodrigo-kiko/sync-granola-tana.git ~/.claude/skills/sync-granola-tana
```

That's it! Claude Code automatically discovers skills in `~/.claude/skills/`.

### Option 2: Manual installation

1. Create the skill directory:

```bash
mkdir -p ~/.claude/skills/sync-granola-tana/references
```

2. Copy the files:

```bash
# Copy from wherever you downloaded them
cp SKILL.md ~/.claude/skills/sync-granola-tana/
cp references/tana-paste-format.md ~/.claude/skills/sync-granola-tana/references/
cp references/sync-logic.md ~/.claude/skills/sync-granola-tana/references/
```

## Usage

In any Claude Code session, type:

```
/sync-granola-tana
```

The skill will:

1. **Verify connectivity** — checks both Granola and Tana MCP servers are reachable
2. **Collect meetings** — fetches the last 7 days from Granola
3. **Check for duplicates** — reads your Tana Daily Pages and matches existing entries
4. **Present a plan** — shows exactly what will be created, updated, or skipped
5. **Ask for confirmation** — nothing is written until you approve
6. **Execute sync** — creates formatted entries in the correct Daily Pages
7. **Report results** — shows a summary of everything that was done

## Configuration

### Timezone

The skill is configured for **America/Sao_Paulo (BRT, UTC-3)** by default. Granola's API returns times in UTC, and the skill converts them to BRT.

To change the timezone, edit the `SKILL.md` file and update:
- The timezone offset in the "Conversao de Horarios" section
- The display format in the Tana Paste template

### Meeting Format

The default Tana Paste format for each meeting is:

```
- HH:MM [Meeting Title] #meeting
  - **Date**: [Day], [Month] [Day]
  - **Attendees**: [Name 1], [Name 2]
  - Summary
    - [Section 1 from Granola summary]
      - [Details as child nodes]
    - [Section 2]
      - [Details]
    - Chat with meeting transcript: https://notes.granola.ai/t/[meeting-id]
```

You can customize this format in `SKILL.md` (Phase 5) and `references/tana-paste-format.md`.

### Deduplication

The skill uses a three-tier matching strategy to avoid duplicates:

| Priority | Method | Confidence |
|----------|--------|------------|
| 1 | **Granola transcript link** in Tana node content | Exact match |
| 2 | **Title/participant overlap** (case-insensitive, min 2 words) | High |
| 3 | **Time-only match** (same HH:MM, different title) | Low — asks user |

The transcript link (`notes.granola.ai/t/{meeting-id}`) is always appended as the last child of the Summary node, ensuring reliable deduplication on future syncs.

## File Structure

```
sync-granola-tana/
├── SKILL.md                          # Main skill definition (6-phase workflow)
├── references/
│   ├── tana-paste-format.md          # Tana Paste syntax reference
│   └── sync-logic.md                 # Matching algorithm & content processing
├── LICENSE                           # MIT License
└── README.md                         # This file
```

### SKILL.md

The main skill file defines:
- **Principles**: Safety rules (never delete, avoid duplicates, confirm before acting)
- **6-phase workflow**: Connectivity → Collect → Check duplicates → Plan → Execute → Report
- **Tana Paste format**: How meetings are structured in Tana
- **Timezone conversion**: UTC to BRT rules
- **Error handling**: What to do when things go wrong

### references/sync-logic.md

Detailed algorithm documentation:
- Date parsing and conversion rules
- Meeting matching algorithm (4-step process)
- Deduplication safety checks
- Summary content processing (Granola markdown → Tana Paste)
- Participant name extraction
- Error recovery procedures

### references/tana-paste-format.md

Tana Paste syntax reference:
- Basic structure and indentation
- Meeting entry format (new and existing)
- Formatting rules for summary content
- What to strip from Granola summaries
- API usage examples
- Character limits and splitting strategies

## Safety Principles

This skill is designed to be safe and predictable:

- **Never deletes content** — Only adds new nodes to Tana
- **Idempotent** — Running multiple times won't create duplicates
- **Confirmation required** — Shows a detailed plan before making any changes
- **Private notes excluded** — Granola private notes are never synced by default
- **Error resilient** — If one meeting fails, continues with the rest and reports failures

## Troubleshooting

### "Granola MCP not accessible"

- Make sure the Granola desktop app is running
- Check that Granola MCP is configured in your Claude Code settings
- Verify the MCP server command is correct

### "Tana MCP not responding on port 8262"

- Open Tana desktop app
- Go to Settings → MCP and make sure it's enabled
- Check that port 8262 is not blocked by a firewall
- Verify your authorization token is correct

### Wrong meeting times

- The Granola API returns times in **UTC**
- The skill converts to BRT (UTC-3) by default
- If your timezone is different, update the offset in `SKILL.md`

### Duplicate entries created

- The skill relies on Granola transcript links for deduplication
- If you manually created meeting entries without links, the skill may not detect them
- In that case, the skill will ask for confirmation when it detects a possible match

## How It Works (Technical Details)

### MCP Communication

- **Granola**: Uses native MCP tools (`list_meetings`, `get_meetings`) via Claude Code's MCP integration
- **Tana**: Uses the local MCP server at `http://127.0.0.1:8262/mcp` with JSON-RPC 2.0 protocol. Tana MCP tools may be available as native Claude Code tools or accessed via HTTP calls as fallback.

### Content Processing Pipeline

1. Granola summary (Markdown with `###` headers, bullets, bold text)
2. Strip timestamps (`[00:52:30]`), horizontal rules (`---`), empty lines
3. Convert `### Header` → `**Header**` (bold Tana node)
4. Preserve bullet hierarchy and bold text
5. Append transcript link as last Summary child
6. Format as Tana Paste (2-space indentation)

### Timezone Handling

```
Granola API: "Feb 19, 2026 2:00 PM" (UTC)
         ↓ subtract 3 hours
Tana entry: "11:00" (BRT)
```

Edge case: If UTC time is before 03:00 AM, the BRT date shifts to the previous day.

## Contributing

Feel free to fork and adapt this skill to your workflow! Some ideas:

- Support different timezones via a config parameter
- Add custom Tana supertag support (beyond `#meeting`)
- Include Granola action items as Tana tasks
- Support filtering meetings by participant or keyword
- Add support for updating existing summaries

## License

MIT
