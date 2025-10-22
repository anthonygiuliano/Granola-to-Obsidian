# Development Plan: Configurable Daily Note Location Setting

## Metadata
```yaml
feature: daily-note-location-setting
status: planned
priority: high
estimated_time: 2-3 hours
created: 2025-10-22
dependencies: []
```

## Problem Statement

The Granola sync plugin currently uses hardcoded heuristics to find daily notes, searching for files containing "Daily" (case-sensitive) in the path with specific date formats. This fails when:
1. Users have lowercase folder names (e.g., `_Inbox/daily/` instead of `Daily/`)
2. Users use custom date formats with day-of-week suffixes (e.g., `2025-10-22-We`)
3. Users store daily notes in non-standard locations

The current implementation in `main.js:1163-1203` cannot reliably locate daily notes for all vault configurations.

## Proposed Solution

Add a configurable setting that allows users to specify their daily note location pattern using moment.js tokens, similar to the Google Calendar plugin's approach. This will make the daily note search deterministic rather than heuristic-based.

## Task Breakdown

### Phase 1: Add Settings Infrastructure (30 min)
**Dependencies:** None

- [ ] **Task 1.1:** Add `dailyNotePath` setting to `DEFAULT_SETTINGS` (line 16-40)
  - Default value: `""` (empty string means use heuristic fallback)
  - Add JSDoc comment explaining moment.js token support

- [ ] **Task 1.2:** Add setting UI in `GranolaSyncSettingTab.display()` (around line 1815)
  - Place in "Daily note integration" section (after line 1826)
  - Add text input with monospace font
  - Placeholder: `e.g., _Inbox/daily/{{YYYY-MM-DD-dd}}`
  - Description: "Optional: Specify exact path pattern to your daily notes using moment.js tokens (YYYY, MM, DD, dd, etc.). Leave empty to use automatic detection."

**Success Criteria:**
- Setting appears in plugin settings UI
- Setting persists correctly in `data.json`
- Placeholder shows example format clearly

### Phase 2: Implement Path Pattern Resolution (45 min)
**Dependencies:** Phase 1

- [ ] **Task 2.1:** Create `resolveDailyNotePath()` helper method
  - Input: date (Date object)
  - Output: resolved file path string
  - Use `window.moment(date).format()` to replace tokens
  - Handle double-brace syntax: `{{YYYY-MM-DD-dd}}` → extract and format

- [ ] **Task 2.2:** Update `getDailyNote()` method (line 1163-1203)
  - Check if `this.settings.dailyNotePath` is configured
  - If configured: use `resolveDailyNotePath()` and `getAbstractFileByPath()`
  - If not configured: fall back to existing heuristic search
  - Add error handling for invalid paths

**Success Criteria:**
- `resolveDailyNotePath()` correctly formats moment.js tokens
- Handles both `{{token}}` and `{token}` syntax
- Falls back gracefully when setting is empty

### Phase 3: Update Periodic Note Detection (30 min)
**Dependencies:** Phase 2

- [ ] **Task 3.1:** Add `periodicNotePath` setting to `DEFAULT_SETTINGS`
  - Default value: `""` (empty string for fallback)

- [ ] **Task 3.2:** Add setting UI for periodic note path (after line 1862)
  - Similar to daily note path setting
  - Placeholder: `e.g., _Inbox/daily/{{YYYY-MM-DD-dd}}`

- [ ] **Task 3.3:** Create `resolvePeriodicNotePath()` helper method
  - Similar to `resolveDailyNotePath()`

- [ ] **Task 3.4:** Update `getPeriodicNote()` method (line 1209-1267)
  - Use new path pattern if configured
  - Keep heuristic fallback

**Success Criteria:**
- Periodic notes can be located using configured path
- Heuristic fallback still works when setting is empty

### Phase 4: Testing & Validation (30 min)
**Dependencies:** Phases 1-3

- [ ] **Task 4.1:** Test with user's actual configuration
  - Path pattern: `_Inbox/daily/{{YYYY-MM-DD-dd}}`
  - Expected file: `_Inbox/daily/2025-10-22-We.md`
  - Verify meeting links are added correctly

- [ ] **Task 4.2:** Test fallback behavior
  - Clear path setting (empty string)
  - Verify heuristic search still works for standard configurations

- [ ] **Task 4.3:** Test edge cases
  - Invalid path patterns
  - Non-existent files
  - Special characters in paths
  - Different date formats

**Success Criteria:**
- Meetings appear in daily note at configured location
- Fallback works when setting is empty
- No errors in console for edge cases

### Phase 5: Documentation (15 min)
**Dependencies:** Phase 4

- [ ] **Task 5.1:** Update `CLAUDE.md` with new settings
  - Document `dailyNotePath` and `periodicNotePath` settings
  - Add examples of valid path patterns
  - Note moment.js token support

- [ ] **Task 5.2:** Add inline code comments
  - Document the path resolution logic
  - Explain fallback behavior

**Success Criteria:**
- `CLAUDE.md` reflects new settings
- Code comments explain the feature clearly

## Implementation Details

### Moment.js Token Support

The setting will support all moment.js format tokens:
- `YYYY` - 4-digit year (2025)
- `MM` - 2-digit month (01-12)
- `DD` - 2-digit day (01-31)
- `dd` - 2-character day of week (Mo, Tu, We, etc.)
- `ddd` - 3-character day of week (Mon, Tue, Wed, etc.)
- `HH`, `mm`, `ss` - Time components (if needed)

### Path Resolution Algorithm

```javascript
resolveDailyNotePath(date) {
  if (!this.settings.dailyNotePath || this.settings.dailyNotePath.trim() === '') {
    return null; // Signal to use fallback
  }

  if (!window.moment) {
    console.error('moment.js not available');
    return null;
  }

  // Replace {{token}} or {token} patterns with moment.js formatting
  let path = this.settings.dailyNotePath;
  const tokenPattern = /\{\{?([^}]+)\}?\}/g;

  path = path.replace(tokenPattern, (match, token) => {
    return window.moment(date).format(token);
  });

  // Ensure .md extension
  if (!path.endsWith('.md')) {
    path += '.md';
  }

  return path;
}
```

## Testing Checklist

- [ ] Setting UI displays correctly
- [ ] Setting persists in `data.json`
- [ ] Path pattern resolves correctly with tokens
- [ ] Daily note is found at configured location
- [ ] Meeting links are added to daily note
- [ ] Fallback heuristic works when setting is empty
- [ ] Works with various date formats (with/without day-of-week)
- [ ] Works with nested folder structures
- [ ] Error handling for invalid patterns
- [ ] Console logging helps debug issues

## Rollback Plan

If issues arise:
1. Revert changes to `main.js`
2. Remove new settings from `DEFAULT_SETTINGS`
3. Remove UI elements from settings tab
4. Plugin will fall back to original heuristic behavior

## Success Metrics

- Daily notes are reliably found regardless of vault structure
- Meeting links appear consistently in daily notes
- No breaking changes to existing functionality
- Clear documentation for users to configure the setting

## Timeline Estimate

- **Phase 1:** 30 minutes
- **Phase 2:** 45 minutes
- **Phase 3:** 30 minutes
- **Phase 4:** 30 minutes
- **Phase 5:** 15 minutes
- **Total:** 2.5 hours (add 30 min buffer = 3 hours)

## Notes

- This feature maintains backward compatibility by using heuristics as fallback
- The Google Calendar plugin's implementation serves as a proven reference
- Moment.js is already available in Obsidian (via `window.moment`)
- The setting is optional, making it non-breaking for existing users
