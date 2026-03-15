---
name: obsidian-knowledge-sync
description: >
  Scan and analyze Obsidian vaults, generate knowledge graphs,
  auto-create MOCs (Map of Contents), detect orphan notes,
  and optimize vault structure for PARA and Zettelkasten methodologies.
  Supports Japanese file names and content.
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Write, Edit, Glob
argument-hint: [command] [options]
context: fork
---

# Obsidian Knowledge Sync

Analyze and optimize Obsidian vaults using PARA and Zettelkasten methodologies.

## Commands

| Command | Description | Runtime |
|---------|-------------|---------|
| `scan` | Vault structure scan with health indicators | 10-30s |
| `links` | Bidirectional link graph analysis | 30-90s |
| `moc` | Auto-generate or update MOC files | 10-30s |
| `orphans` | Detect orphan notes with classification | 30-60s |
| `context` | Inject relevant vault knowledge for a query | 15-40s |
| `para` | PARA method placement advisor | 10-30s |

## Global Options

| Option | Default | Description |
|--------|---------|-------------|
| `--vault-path PATH` | cwd | Root path of the Obsidian vault |
| `--language [ja\|en]` | `ja` | Language for generated content |
| `--dry-run` | false | Preview without writing files |

## Japanese File Name Handling

All commands MUST: use double-quoted paths in shell commands, never URL-encode
Japanese characters, display Japanese filenames as-is, use UTF-8 for all I/O.

## Security: Input Validation (MANDATORY — run before every command)

### Step 0: Validate all path inputs

Before executing ANY command, validate `--vault-path`, `--scope`, `--target`, `--analyze`, `--batch`:

1. **Reject shell metacharacters**: If the path contains any of `` ` `` `$` `;` `|` `&` `(` `)` `\n`, STOP and error: "Path contains invalid characters."
2. **Resolve to absolute path**: Use `realpath` or equivalent. Reject symlinks pointing outside the vault.
3. **Verify vault**: The resolved `--vault-path` MUST contain a `.obsidian/` directory. If not, error: "Not an Obsidian vault."
4. **Scope containment**: `--scope`, `--target`, `--batch`, `--analyze` paths MUST be descendants of the resolved vault-path. If not, error: "Path is outside the vault."
5. **Write containment**: All Write/Edit operations MUST target paths that are descendants of the resolved vault-path. Never write outside the vault.

### Sensitive file denylist

Skip files matching these patterns silently (do not read, grep, or include in output):
extensions `.pem`, `.key`, `.p12`, `.pfx`, and filenames starting with `.env`.

### Query sanitization

For `--query` input (context command): strip `/`, `..`, and escape regex metacharacters before embedding in Grep/Glob patterns.

### Tool restrictions

This skill does NOT use Bash. All file operations use Glob (discovery), Grep (search),
Read (content), Write/Edit (MOC generation). File metadata (modification dates) is
extracted by reading the first lines of each file for date patterns in the filename
(e.g., `YYYYMMDD_` prefix) or frontmatter `date:` field.

---

## Command: scan

Scan vault folder structure to depth N and produce a structure report.

**Options**: `--depth N` (default 3), `--exclude PATTERN` (default `.obsidian,node_modules,.git`)

### Steps

1. **Validate vault**: `Glob: ${VAULT_PATH}/**/*.md`. If 0 files found, error: "No markdown files found. This may not be an Obsidian vault."

2. **Collect folder stats** for each top-level folder up to `--depth`:
   - File count: `Glob: ${VAULT_PATH}/FOLDER/**/*.md` then count results
   - Subfolder count: `Glob: ${VAULT_PATH}/FOLDER/*/` then count results

3. **Detect methodology**:
   - PARA: `Glob: ${VAULT_PATH}/[0-9][0-9]_*/` -- look for `*Inbox*`, `*Memo*`, `*Project*`, `*Output*`, `*Memory*`, `*Archive*` patterns
   - Zettelkasten: `Grep: pattern="^[0-9]{8,14}" --glob="*.md" --output_mode=count`

4. **Compute health indicators**:

| Indicator | Warning Threshold |
|-----------|-------------------|
| Inbox backlog | > 20 files |
| Memo backlog | > 10 files |
| Orphan ratio (estimate) | > 30% |
| Stale ratio (90+ days) | > 40% |

5. **Output report** with sections: Overview (total files, folders, methodology), Folder Breakdown table (Folder | .md Files | Subfolders), Health Indicators table (Indicator | Value | Status).

**Errors**: Path not found -> print attempted path. Permission denied -> skip folder, note in footer. Over 10,000 files -> warn, suggest `--depth 2`.

---

## Command: links

Analyze `[[bidirectional links]]` across a scoped folder or full vault.

**Options**: `--scope FOLDER` (recommended), `--min-links N` (default 0), `--top N` (default 10)

**Scope warning**: On vaults with 4,000+ files, full-vault analysis requires many tool calls. Always recommend `--scope`. If user omits it, warn and ask confirmation.

### Steps

1. **Extract wikilinks**:
```bash
Grep: pattern="\[\[([^\]|]+)(\|[^\]]+)?\]\]" path="${SCOPE}" --glob="*.md" --output_mode=content
```
Parse: `[[Name]]` -> target="Name", `[[Name|Alias]]` -> target="Name".

2. **Build link index**: Create `out_links[source]` and `in_links[target]` maps.

3. **Hub notes**: Sort by out-link count descending, show top N.

4. **Broken link detection**: A link is broken if target basename has no matching .md file (case-insensitive). Exclusion rules -- classify as "External Reference" instead of broken:
   - Target starts with `.claude/` (system reference)
   - Target starts with `../` (relative path)
   - Target contains `/` pointing outside vault

5. **Output report** with sections: Scope info, Network Statistics (total links, unique targets, avg links/note), Hub Notes table (Rank | Note | Out-Links | In-Links | Folder), Broken Links table, External References table.

**Errors**: No `[[]]` found -> "No wikilinks found. Vault may use standard markdown links." Scope not found -> print path.

---

## Command: moc

Auto-generate or update a MOC (Map of Contents) file for a target folder.

**Options**: `--target FOLDER` (required), `--style [flat|hierarchical]` (default hierarchical), `--update`, `--dry-run`

### Steps

1. **Scan target**: `Glob: ${TARGET}/**/*.md`. Exclude `_*-MOC.md` and `.obsidian/`.

2. **Extract metadata** from each file (first 10 lines): title (first `# ` heading or filename), tags, last modified date (`stat`).

3. **Organize**: `hierarchical` groups by subfolder, `flat` is alphabetical.

4. **Generate MOC**: Read `reference/moc-templates.md` for language-appropriate template. MOC filename: `_[FolderBaseName]-MOC.md` (e.g., `04_Memory/AI/` -> `_AI-MOC.md`).

5. **Idempotency**: Auto-generated sections MUST be wrapped in:
```
<!-- obsidian-knowledge-sync:start -->
(generated content)
<!-- obsidian-knowledge-sync:end -->
```
On `--update`: replace ONLY content between markers, preserve everything outside. If existing MOC has no markers: do NOT overwrite, warn user, show `--dry-run` output.

6. **Write**: `--dry-run` prints to stdout. Otherwise write to `${TARGET}/_[Name]-MOC.md`.

**Errors**: Empty folder -> "No .md files found." No target -> print path error. Existing MOC without markers -> warning + dry-run preview. `--update` with no existing MOC -> create new.

---

## Command: orphans

Detect notes with zero in-links and classify them into 3 categories.

**Options**: `--scope FOLDER` (default full vault), `--exclude PATTERN`

### Default Exclusions (always excluded)

`**/06_Templates/**`, `**/02_Daily/**`, `**/_*-MOC.md`, `**/99_Archive/**`, `**/.obsidian/**`, `**/07_System/**`, `**/08_prompts/**`

### Steps

1. **Build in-link map**: Reuse `links` command logic. For each `[[target]]`, record target as having >= 1 in-link.

2. **Find orphans**: `Glob: ${SCOPE}/**/*.md`. File is orphan if its basename (without .md) is absent from in-link map and not excluded.

3. **Classify** each orphan:

| Category | Rule | Action |
|----------|------|--------|
| **Structural** | Has 3+ out-links OR modified > 180 days ago | Normal; suggest MOC addition |
| **Review** | Has tags AND modified within 180 days AND < 3 out-links | User should review |
| **Integrate** | All other cases (recent, no tags, no out-links) | Needs links to related notes |

Classification priority: check Structural first, then Review, then Integrate.

4. **Output report** with sections: Summary (total scanned, excluded, orphan count + %), Classification table, Orphans by Folder table, Integration Candidates (top 20).

**Errors**: Scope not found -> print path. No orphans -> "No orphan notes found. Your vault is well-connected."

---

## Command: context

Find vault notes relevant to a query and output a compact context block.

**Options**: `--query TEXT` (required), `--max-tokens N` (default 3000), `--strategy [recent|relevant|hub]` (default relevant), `--max-notes N` (default 5)

### Strategies

| Strategy | Criteria | Use Case |
|----------|----------|----------|
| `recent` | Sort by last-modified desc | "What have I worked on?" |
| `relevant` | Keyword match in title+tags+headings | Topical queries |
| `hub` | Prefer high link-count notes | Broad context |

### Steps

1. **Extract keywords** from query. Remove English stop words (the, a, is, are, of, for, etc.). For Japanese, split on particles (wa, ga, no, wo, ni, de, to).

2. **Search**:
```bash
Glob: **/*{keyword}*.md
Grep: pattern="(#keyword|## .*keyword)" --glob="*.md" --output_mode=files_with_matches
```

3. **Score** each match:

| Factor | Points |
|--------|--------|
| Title contains keyword | +3/keyword |
| Tag matches keyword | +2/keyword |
| Heading contains keyword | +1/keyword |
| Modified < 7 days ago | +2 (recent strategy) |
| Link count > 10 | +1 (hub strategy) |

4. **Build context**: For top N notes, extract title, path, tags, first 200 chars. Estimate tokens: 1 token ~ 3 chars (English) or 1.5 chars (Japanese). Stay within `--max-tokens`.

5. **Output**: For each note: `### N. [[Title]] (score pts)` with Path, Tags, Preview. Append Vault Quick Stats (active projects count, inbox items, daily note status).

**Errors**: No matches -> "No notes matching '[query]'. Try broader keywords." Empty query -> show usage example.

---

## Command: para

Analyze files against the PARA method and suggest optimal placement.

**Options**: `--analyze FILE` or `--batch FOLDER` (one required)

### Steps

1. **Load rules**: Read `reference/para-rules.md` for decision tree and folder mappings.

2. **Extract signals** from each file:
   - Deadline keywords: `deadline`, `due`, `期限`, `〆切`, `締切`
   - Active tasks: count `- [ ]` unchecked checkboxes
   - Status tags: `#status/*`, `#priority/*`, `#project`, `#area`
   - Modification date, link count, current folder location

3. **Apply decision tree**:
```
Deadline/end goal? -> YES -> Projects (completed? -> Archive)
                   -> NO  -> Ongoing responsibility? -> YES -> Areas
                                                     -> NO  -> Reference info? -> YES -> Resources
                                                                               -> NO  -> Irrelevant? -> Archive, else Inbox
```
Full tree with signal weights in `reference/para-rules.md`.

4. **Compare** current location vs recommendation. If same, mark "Correctly placed".

5. **Output**:
   - Single file: Current location, Recommended category + path, Confidence (High/Medium/Low), Reason, Detected signals list.
   - Batch: Summary (files analyzed, correctly placed %, misplaced %), Misplaced Files table (File | Current | Recommended | Confidence | Reason), Inbox Review Queue (files in Inbox for 7+ days).

**Errors**: File not found -> print path. Neither option given -> show usage. Empty folder -> "No .md files."

---

## Shared Error Handling

1. **Path errors**: Print exact attempted path, suggest checking for typos.
2. **Empty results**: Never output empty reports. Always state "No [items] found" with context.
3. **Large vaults (10,000+)**: Warn and recommend `--scope` / `--depth 2`.
4. **Permission errors**: Skip inaccessible files, note in report footer.
5. **Encoding errors**: Skip non-UTF-8 files, note filename in errors section.

## Privacy Notice

This skill reads and analyzes personal vault content. Output previews (e.g., context
command) contain excerpts from your notes. These outputs are visible to the LLM provider
during skill execution. Do not use this skill on vaults containing highly sensitive
data (medical records, financial credentials, etc.) without understanding this risk.

## Performance Guidelines

| Vault Size | Approach |
|------------|----------|
| < 1,000 | Full vault operations fine |
| 1,000 - 5,000 | Use `--scope` for `links` and `orphans` |
| 5,000 - 10,000 | `--scope` for all commands, `--depth 2` for scan |
| > 10,000 | Always scope + limit depth |

## References

- `reference/para-rules.md` - PARA decision tree and folder mapping rules
- `reference/moc-templates.md` - MOC templates (ja/en) with idempotency markers
