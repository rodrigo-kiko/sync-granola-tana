# Sync Logic Reference

Detailed algorithm for matching Granola meetings to Tana Daily Page entries.

---

## Date Parsing

### Granola date format

Input: `"Feb 19, 2026 11:00 AM"` or `"Feb 19, 2026 2:00 PM"`

Parse to:
- **ISO date**: `2026-02-19` (for Tana calendar API)
- **24h time**: `11:00` or `14:00` (for display)

### Conversion rules

- `12:00 PM` = `12:00` (noon)
- `12:00 AM` = `00:00` (midnight)
- `1:00 PM` = `13:00`
- `11:00 AM` = `11:00`

Pad single-digit hours: `2:00 PM` -> `14:00`

---

## Meeting Matching Algorithm

### Step 1: Normalize meeting title

From Granola title, extract:
- Raw title: `"1:1 Geison e Rodrigo Pinto"`
- Normalized: lowercase, trimmed, remove extra spaces

### Step 2: Check Tana Daily Page children

For each child node in the Daily Page:
1. Read node title
2. Try to extract time prefix (pattern: `HH:MM` at start)
3. Extract remaining title after time prefix

### Step 3: Match criteria (ANY of these = match)

**Time + Title match** (strongest):
- Tana node starts with same `HH:MM`
- AND remaining title contains key words from Granola title
- Example: Tana `"11:00 1:1 Geison"` matches Granola `"1:1 Geison e Rodrigo Pinto"`

**Title-only match** (fallback):
- Tana node title contains the Granola meeting title (case-insensitive)
- OR Granola title contains the Tana node title
- Minimum 3 words overlap to avoid false positives

**Time-only match** (weakest, needs confirmation):
- Same `HH:MM` but different titles
- Ask user to confirm if it's the same meeting

### Step 4: Classification

- **MATCH_EXACT**: Time + title match -> auto-sync
- **MATCH_PROBABLE**: Title-only or time-only match -> show to user, ask confirmation
- **NO_MATCH**: New meeting -> create

---

## Deduplication Safety

Before creating a new entry, double-check:

1. No existing child with same time prefix
2. No existing child with > 50% word overlap in title
3. If either condition is true, flag as potential duplicate and ask user

---

## Summary Content Processing

### Input (Granola summary markdown)

```markdown
### Section Title

- Bullet point 1
- Bullet point 2
  - Sub-bullet

**Key point**: Value text

Some paragraph text about the meeting.
```

### Output (Tana Paste)

```
- **Section Title**
  - Bullet point 1
  - Bullet point 2
    - Sub-bullet
  - **Key point**: Value text
  - Some paragraph text about the meeting.
```

### Processing rules

1. `### Header` -> `- **Header**` (bold node)
2. `#### Sub-header` -> `  - **Sub-header**` (indented bold node)
3. `- bullet` -> preserve as `- bullet`
4. `  - sub-bullet` -> preserve indentation
5. Bare paragraph -> `- paragraph text`
6. `**bold**: text` -> preserve as-is
7. Remove `[HH:MM:SS]` timestamps
8. Remove `---` horizontal rules
9. Remove empty lines (don't create empty nodes)
10. Collapse multiple consecutive empty sections

### CRITICAL: Content fidelity rules

These rules override any tendency to summarize, curate, or simplify content:

**Rule 1 - Preserve all Unicode characters (accents, cedillas, special chars)**:
- The Granola summary contains text in the original language with proper accents
- NEVER strip or normalize accents (á, é, í, ó, ú, ã, õ, ç, â, ê, etc.)
- Tana fully supports Unicode — pass the text through as-is
- Example: "Análise do padrão de rigidez" must stay exactly as-is
- WRONG: "Analise do padrao de rigidez" (accents stripped)

**Rule 2 - Import ALL bullets with zero omissions**:
- Every single bullet in the Granola summary MUST appear in the Tana Paste output
- Do NOT curate, select, summarize, or skip any bullets
- Do NOT decide that certain bullets are "less important" or "redundant"
- If Granola has 30 bullets under a section, Tana gets 30 nodes
- The skill's job is FAITHFUL TRANSFER, not editorial curation

**Rule 3 - Preserve full text of each bullet (no truncation)**:
- Copy the complete text of each bullet, including any prefix labels
- If a bullet starts with "Foco:", "Tema:", "Expectativas:", etc., keep those prefixes
- Do NOT treat label prefixes as metadata to extract — they are part of the content
- Example: "**Foco:** Reconhecimento do método como proteção" → keep exactly as-is
- WRONG: "Reconhecimento do metodo como protecao" (prefix removed + accents stripped)

**Rule 4 - Preserve full hierarchy (no flattening)**:
- If Granola has a bullet "Fatos da semana:" with sub-bullets underneath, maintain that hierarchy
- Do NOT flatten nested structures into a single level
- The indentation in Tana Paste (2 spaces per level) must reflect the original nesting

### Participant extraction

From Granola `known_participants`:
- Extract names (before email/org info)
- Remove note creator marker "(note creator)"
- Remove email addresses
- Join with ", "

Example:
```
Input: "Rodrigo Pinto (note creator) from SocialPrompts.ai <rodrigo.pinto@stage2sales.com>, Geison_isidro <geison_isidro@hotmail.com>"
Output: "Rodrigo Pinto, Geison_isidro"
```

---

## Error Recovery

### Partial failure handling

If importing meeting 3 of 5 fails:
1. Log the error with meeting title and error message
2. Continue with meetings 4 and 5
3. Report partial success in final report
4. Suggest retrying failed meetings

### Tana Paste size limits

If a single meeting summary is very large:
1. Try full import first
2. If fails, split by top-level sections
3. Import each section as separate `import_tana_paste` call
4. All targeted to same meeting node
