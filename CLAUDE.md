# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Granola Sync is an Obsidian plugin (v1.6.3) that syncs meeting notes from Granola AI to Obsidian vaults. It's a **single-file plugin** (main.js, ~2000 lines) that communicates with the Granola API using locally stored authentication credentials.

**Key characteristics:**
- Desktop-only Obsidian plugin (requires Obsidian v1.6.6+)
- Pre-compiled JavaScript (no TypeScript or build pipeline in production)
- Monolithic architecture with clear separation of concerns
- MIT licensed by Danny McClelland

## Development Commands

Since this is a pre-built plugin with no package.json or build scripts in the repository, development typically involves:

1. **Testing changes**: Copy `main.js`, `manifest.json`, and `styles.css` to `.obsidian/plugins/granola-sync/` in a test vault
2. **Reloading plugin**: In Obsidian, disable and re-enable the plugin in Settings → Community Plugins
3. **Debugging**: Open Obsidian's Developer Console (Cmd/Ctrl + Shift + I) to view logs and errors

**Note**: The README mentions `npm install` and `npm run build` for development, but these files are not present in the current repository. The distributed version uses pre-compiled `main.js`.

## Architecture

### Single-File Structure

All plugin code lives in `main.js`:

**GranolaSyncPlugin (Plugin Class)**
- Lines 43-79: `onload()` - Initializes ribbon icon, commands, settings, and auto-sync
- Lines 219-316: `syncNotes()` - Main orchestrator for sync process
- Lines 318-374: `loadCredentials()` - Reads Granola auth from filesystem
- Lines 376-462: API methods (`fetchGranolaDocuments`, `fetchGranolaFolders`, `fetchTranscript`)
- Lines 464-563: Content conversion (ProseMirror → Markdown, transcripts)
- Lines 636-684: File organization (date-based and folder-based paths)
- Lines 686-842: Deduplication (`findExistingNoteByGranolaId`, `findDuplicateNotes`)
- Lines 844-1029: `processDocument()` - Core note creation/update logic
- Lines 1042-1256: Daily Notes and Periodic Notes integration
- Lines 1326-1496: Metadata generation (tags, URLs, attendees, folders)

**GranolaSyncSettingTab (Settings UI Class)**
- Lines 1505-1991: Complete settings interface with sections for sync, metadata, integrations, and experimental features

### Sync Flow

```
syncNotes()
  ↓
Load credentials from filesystem (supabase.json)
  ↓
Fetch documents from Granola API (/v2/get-documents)
  ↓
If folder support enabled: Fetch folders (/v1/get-document-lists-metadata)
  ↓
For each document:
  ├─ Find existing note by granola_id (configurable search scope)
  ├─ If exists: Update metadata OR skip (based on settings)
  └─ If new: Create with templated filename
  ↓
Update Daily Note and/or Periodic Note with today's meetings
  ↓
Update status bar
```

### Authentication System

The plugin reads authentication credentials from the Granola desktop app's local storage:

**Default paths:**
- macOS: `~/Library/Application Support/Granola/supabase.json`
- Windows: `AppData/Roaming/Granola/supabase.json`
- Linux: `~/.config/Granola/supabase.json`

**Credential loading logic** (lines 318-374):
1. Tries user-configured path first
2. Falls back to platform-specific defaults
3. Extracts access token from either `workos_tokens` or `cognito_tokens`
4. Returns token for API authorization

### API Endpoints

**Base URL**: `https://api.granola.ai`

- `/v2/get-documents` - Fetch all documents (limit: 100, offset: 0)
- `/v1/get-document-lists-metadata` - Fetch folder mappings
- `/v1/get-document-transcript` - Fetch transcript for document_id

All requests use `Authorization: Bearer {token}` header.

### File Organization Strategies

The plugin supports two independent organization modes:

**Date-based folders** (lines 636-654):
- Creates folder hierarchy like `2025/10/21/`
- Format configurable via `dateFolderFormat` setting
- Enabled with `enableDateBasedFolders`

**Granola folder mirroring** (lines 656-684):
- Mirrors folder structure from Granola
- Maps documents to folders using `documentToFolderMap`
- Enabled with `enableGranolaFolders`

### Deduplication & Search Scopes

**Experimental feature** (lines 686-842): Configurable search scope for finding existing notes by `granola_id`:

- `syncDirectory` (default): Only searches sync directory
- `entireVault`: Searches all markdown files in vault
- `specificFolders`: Searches only specified folders

This allows users to move notes around the vault without creating duplicates.

### Path Pattern Resolution (v1.6.4+)

The plugin supports configurable path patterns for locating daily and periodic notes using moment.js tokens.

**Path Pattern Settings**:
- `dailyNotePath`: Path pattern for daily notes (default: empty string for heuristic fallback)
- `periodicNotePath`: Path pattern for periodic notes (default: empty string for heuristic fallback)

**Supported Token Syntax**:
- Double braces: `{{YYYY-MM-DD-dd}}` (recommended)
- Single braces: `{YYYY-MM-DD-dd}` (also supported)

**Common moment.js tokens**:
- `YYYY` - 4-digit year (2025)
- `MM` - 2-digit month (01-12)
- `DD` - 2-digit day (01-31)
- `dd` - 2-character day of week (Mo, Tu, We, Th, Fr, Sa, Su)
- `ddd` - 3-character day of week (Mon, Tue, Wed, etc.)
- `MMMM` - Full month name (October)
- `D` - Day without leading zero (1-31)

**Example patterns**:
- `_Inbox/daily/{{YYYY-MM-DD-dd}}` → `_Inbox/daily/2025-10-22-We.md`
- `Daily Notes/{{YYYY}}/{{MM}}/{{DD}}` → `Daily Notes/2025/10/22.md`
- `Journal/{{MMMM}} {{D}}, {{YYYY}}` → `Journal/October 22, 2025.md`

**Implementation details**:
- Path resolution methods: `resolveDailyNotePath()` and `resolvePeriodicNotePath()` (lines 594-654)
- Uses `window.moment(date).format()` for token replacement
- Automatically appends `.md` extension if missing
- Returns `null` if pattern is empty or moment.js is unavailable, triggering fallback to heuristic search
- Logs informational message to console when falling back

### Daily Notes & Periodic Notes Integration

**Daily Notes** (lines 1227-1279):
- **Configurable path pattern** (v1.6.4+): Uses `dailyNotePath` setting with moment.js tokens if configured
- Falls back to date-based heuristics (multiple formats supported) if path is not configured
- Path resolution via `resolveDailyNotePath()` method (lines 594-623)
- Supports patterns like `_Inbox/daily/{{YYYY-MM-DD-dd}}` → `_Inbox/daily/2025-10-22-We.md`
- Updates "## Granola Meetings" section with meeting links and times
- Format: `- HH:MM [[filepath|title]]`

**Periodic Notes** (lines 1285-1352):
- **Configurable path pattern** (v1.6.4+): Uses `periodicNotePath` setting with moment.js tokens if configured
- Falls back to heuristics if not configured
- Path resolution via `resolvePeriodicNotePath()` method (lines 625-654)
- Detects if Periodic Notes plugin is available
- Uses similar logic to Daily Notes with separate section heading
- Can be enabled independently from Daily Notes

Both integrations:
1. Collect today's meetings during sync
2. Find or create daily note
3. Update or append meetings section
4. Preserve other content in the note

### Metadata Generation

**Attendee tags** (lines 1326-1398):
- Extracts from Granola `people` field, Google Calendar, or email addresses
- Converts to tags: "John Smith" → `person/john-smith`
- Template customizable via `attendeeTagTemplate` setting
- Filters out user's own name via `myName` setting

**Folder tags** (lines 1400-1432):
- Creates tags from Granola folder names
- Template: `folder/{name}` (customizable)

**Granola URLs** (lines 1434-1442):
- Generates `https://notes.granola.ai/d/{document_id}` links
- Added to frontmatter as `granola_url`

### Content Conversion

**ProseMirror to Markdown** (lines 464-563):
- Converts Granola's ProseMirror JSON to Markdown
- Handles: headings, paragraphs, bullet lists, nested lists, hard breaks
- Recursive processing for nested structures
- Preserves indentation for list items

**Transcript formatting** (lines 565-585):
- Converts segments to `[HH:MM:SS] Speaker: text` format
- Maps "microphone" → "Me", "system" → "Them"

## Version Management & Release Process

**Critical rules from .claude_rules:**

1. **Version tags**: Use numbers only, NO "v" prefix (e.g., `1.6.2`, NOT `v1.6.2`)
2. **Release process**: Git tag with version number triggers GitHub Actions
3. **GitHub Actions**: Automatically creates release and attaches assets (`main.js`, `manifest.json`, `styles.css`)
4. **Version synchronization**: Release tag MUST exactly match `manifest.json` version
5. **Manual uploads**: Never manually upload release assets - GitHub Actions handles this

**When bumping version:**
1. Update `manifest.json` version field
2. Update `versions.json` with new entry
3. Create git tag with version number (no "v" prefix)
4. Push tag to trigger automated release

**Commit messages:**
- Do NOT include Claude Code attribution
- Keep messages clean and professional
- Focus on actual changes

## Settings Architecture

**Default settings** (lines 16-40) include:
- `syncDirectory`: Target folder in vault
- `filenameTemplate`: Template with variables like `{title}`, `{created_date}`, `{id}`
- `dateFormat`: Custom format using tokens (YYYY, MM, DD, HH, mm, ss)
- `autoSyncFrequency`: Interval in milliseconds (or 0 for disabled)
- `skipExistingNotes`: Preserve manual edits by not updating existing notes
- `includeAttendeeTags`, `includeFolderTags`, `includeGranolaUrl`: Metadata options
- `existingNoteSearchScope`: Experimental search scope configuration
- `enableDateBasedFolders`, `enableGranolaFolders`: Organization modes
- `dailyNotePath`: Optional path pattern for daily notes using moment.js tokens (e.g., `_Inbox/daily/{{YYYY-MM-DD-dd}}`)
- `periodicNotePath`: Optional path pattern for periodic notes using moment.js tokens
- Integration toggles for Daily Notes and Periodic Notes

Settings are:
- Loaded via `this.loadData()` in `onload()` (merged with defaults)
- Persisted via `this.saveData(this.settings)`
- Two save methods: `saveSettings()` (restarts auto-sync) and `saveSettingsWithoutSync()` (no restart)

## Known Issues & Workarounds

**Toggle bug workaround** (v1.6.3):
- Granola folders feature had toggle issues
- Replaced with button-based approach (lines 1868-1897)
- Button shows "Enable Granola Folders" or "Disable Granola Folders" based on state

**Periodic Notes API**:
- Periodic Notes plugin API not directly accessible
- Uses heuristic file search instead (similar to Daily Notes approach)
- Searches for files matching today's date in various formats

## Code Navigation Quick Reference

| Feature | Lines |
|---------|-------|
| Plugin lifecycle | 43-83 |
| Main sync orchestration | 219-316 |
| Credential loading | 318-374 |
| API calls | 376-462 |
| Content conversion | 464-563 |
| Date formatting | 565-593 |
| Path pattern resolution | 594-654 |
| File organization | 656-715 |
| Deduplication | 717-873 |
| Document processing | 875-1060 |
| Daily Notes integration | 1227-1279 |
| Periodic Notes integration | 1285-1352 |
| Metadata generation | 1357-1527 |
| Settings UI | 1536-2022 |

## Important Patterns

**Error handling**: Every async operation wrapped in try-catch with:
- Graceful fallbacks (return null/false/empty array)
- Status bar error messages
- Console logging for debugging

**Status bar updates**: Real-time feedback during sync:
- "Idle" → "Syncing..." → "X notes synced" or "Error - [details]"
- Temporary success/error messages (3-5 seconds)

**Frontmatter structure**:
```yaml
---
granola_id: abc123def456
title: "Meeting Title"
granola_url: "https://notes.granola.ai/d/abc123def456"
created_at: 2025-10-21T14:30:00.000Z
updated_at: 2025-10-21T15:45:00.000Z
tags:
  - person/john-smith
  - folder/work-meetings
---
```

**Template variables** (used in filename and tag templates):
- `{title}`, `{id}`, `{name}`
- `{created_date}`, `{updated_date}`, `{created_time}`, `{updated_time}`
- `{created_datetime}`, `{updated_datetime}`

## Testing Considerations

When making changes:
1. Test with auto-sync disabled first (manual sync only)
2. Use status bar for real-time feedback
3. Check console for detailed logs and errors
4. Test with "Skip Existing Notes" enabled to avoid overwriting test data
5. Use "Find Duplicate Notes" utility to verify deduplication logic
6. Test Daily/Periodic Notes integration with actual daily note files
7. Verify frontmatter parsing with notes that have complex metadata
