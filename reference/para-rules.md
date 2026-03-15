# PARA Method Decision Rules

This reference file contains the complete PARA decision tree and folder mapping
rules used by the `para` command.

## PARA Categories

| Category | Purpose | Lifespan | Key Signal |
|----------|---------|----------|------------|
| **Projects** | Tasks with a deadline and defined outcome | Finite | Has deadline or end goal |
| **Areas** | Ongoing responsibilities to maintain | Indefinite | Recurring, no end date |
| **Resources** | Reference information for future use | Long-term | Informational, not actionable |
| **Archive** | Completed or inactive items | Permanent | No longer active or relevant |

## Folder Mapping Patterns

The skill auto-detects PARA folder structure using these patterns:

### Standard PARA (no prefix)
```
Projects/
Areas/
Resources/
Archive/
```

### Numbered Prefix PARA (common in Obsidian)
```
00_Memo/      -> Capture (pre-Inbox)
01_Inbox/     -> Triage queue
02_Daily/     -> Chronological (excluded from PARA analysis)
03_Input/     -> Short-term Resources (active reference)
04_Memory/    -> Long-term Resources
05_Output/    -> Projects + Areas
  Projects/@Active/    -> Active Projects
  Projects/@Planning/  -> Planning Projects
  Projects/@Completed/ -> Projects ready for Archive
  Areas/               -> Areas (ongoing responsibilities)
06_Templates/ -> Excluded (system)
07_System/    -> Excluded (system)
08_prompts/   -> Excluded (system)
99_Archive/   -> Archive
```

### Detection Logic

```
1. Check for numbered prefix folders (NN_Name pattern)
   -> If found: Use numbered prefix mapping
2. Check for standard PARA folder names
   -> If found: Use standard mapping
3. Fallback: Treat top-level folders as custom categories
   -> Report structure without PARA-specific recommendations
```

## Decision Tree (Full)

```
START: Analyze file content and metadata
  |
  |-- Step 1: Is this a system/template file?
  |     Located in Templates/, System/, .obsidian/, prompts/?
  |     YES -> EXCLUDED (not part of PARA flow)
  |     NO  -> continue
  |
  |-- Step 2: Is this a daily/chronological note?
  |     Located in Daily/ or matches YYYY-MM-DD pattern?
  |     YES -> EXCLUDED (chronological, not PARA)
  |     NO  -> continue
  |
  |-- Step 3: Has an active deadline or defined end goal?
  |     Signals (any match = YES):
  |       - Contains "deadline", "due date", "due by", "target date"
  |       - Contains Japanese: "期限", "〆切", "締切", "目標日"
  |       - Has #project tag
  |       - Has unchecked tasks (- [ ]) AND a date reference
  |       - Located in Projects/ folder
  |     YES -> Step 3a
  |     NO  -> Step 4
  |
  |-- Step 3a: Is the project completed?
  |     Signals:
  |       - All checkboxes checked (no - [ ] remaining)
  |       - Has #status/completed tag
  |       - Located in @Completed/ subfolder
  |       - Not modified in 30+ days AND has "completed" in content
  |     YES -> ARCHIVE (99_Archive/)
  |     NO  -> PROJECTS (05_Output/Projects/@Active/)
  |
  |-- Step 4: Is this an ongoing responsibility or standard?
  |     Signals (any match = YES):
  |       - Contains "recurring", "ongoing", "maintain", "standard"
  |       - Contains Japanese: "継続", "運用", "管理", "定期"
  |       - Has #area tag
  |       - Modified regularly (multiple modifications in last 90 days)
  |       - Located in Areas/ folder
  |     YES -> AREAS (05_Output/Areas/)
  |     NO  -> Step 5
  |
  |-- Step 5: Is this useful reference information?
  |     Signals (any match = YES):
  |       - Has informational tags (#reference, #guide, #manual, #howto)
  |       - Contains structured knowledge (3+ headings, lists, code blocks)
  |       - Has 2+ incoming links (other notes reference it)
  |       - Contains "reference", "guide", "manual"
  |       - Contains Japanese: "参考", "ガイド", "マニュアル", "手順"
  |     YES -> RESOURCES (04_Memory/)
  |     NO  -> Step 6
  |
  |-- Step 6: Is it no longer relevant?
  |     Signals:
  |       - Not modified in 90+ days
  |       - Zero incoming and outgoing links
  |       - No active tasks
  |       - Content references past dates or deprecated tools
  |     YES -> ARCHIVE (99_Archive/)
  |     NO  -> INBOX (01_Inbox/) for manual triage
```

## Confidence Scoring

| Confidence | Criteria |
|------------|----------|
| **High** | 3+ signals match a single category, no conflicting signals |
| **Medium** | 2 signals match, or signals point to 2 categories |
| **Low** | Only 1 signal match, or strong conflicting signals |

When confidence is Low, always present the analysis to the user
rather than making an automatic recommendation.

## Special Cases

### Files in 03_Input/ (Short-term Resources)
- Files in `03_Input/` are "active reference" -- they are Resources
  that are currently being used
- If a file in `03_Input/` has not been accessed in 30+ days,
  suggest moving to `04_Memory/` (long-term) or `99_Archive/`

### Files in 00_Memo/ (Quick Capture)
- Memos are pre-triage by definition
- The `para` command should suggest moving ALL memos to `01_Inbox/`
  for proper PARA classification
- Never suggest moving directly from `00_Memo/` to a PARA category

### Files in 01_Inbox/
- Inbox files older than 7 days are "stale inbox" items
- Batch analysis of `01_Inbox/` should flag stale items prominently

### MOC Files (_*-MOC.md)
- MOC files belong in their parent category folder
- They should NOT be recommended for Archive even if stale
- They are infrastructure, not content

## Mapping Output Paths

When recommending a move, provide the specific target path:

| Recommendation | Target Path Pattern |
|---------------|-------------------|
| Projects (active) | `05_Output/Projects/@Active/[project-name]/` |
| Projects (planning) | `05_Output/Projects/@Planning/` |
| Areas | `05_Output/Areas/[area-name]/` |
| Resources (long-term) | `04_Memory/[category]/` |
| Resources (short-term) | `03_Input/[category]/` |
| Archive | `99_Archive/[year]/` |
| Inbox (triage) | `01_Inbox/` |

For the `[category]` placeholder, detect the best subcategory from existing
folder names in the target. If no match, suggest the most relevant existing subfolder
or recommend creating a new one.
