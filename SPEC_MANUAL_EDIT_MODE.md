# Specification: Allow Manual Editing in Accept/Reject Mode

## Overview

This spec defines the implementation of an optional "Allow Manual Edit" feature that enables users to continue typing and editing text while navigating through AI-generated corrections. Currently, the editor enters a "locked" mode during corrections where the textarea becomes read-only and all keyboard input is routed to correction navigation/acceptance.

## Feature Requirements

### 1. UI Toggle Control

Add an "Allow Manual Edit" toggle button in the correction controls section:

- **Placement**: Inside the correction controls that appear during correction mode (with prev/next buttons)
- **Visibility**: Only visible when corrections are active, making the feature discoverable in context
- **Default State**: OFF (preserves current behavior)
- **Visual Design**: Checkbox or toggle switch style to differentiate from navigation buttons
- **Active State**: Clear visual indication when ON (checked state with label "Manual editing enabled")
- **Persistence**: Store state in localStorage as `allowManualEdit` boolean
- **Accessibility**: ARIA pressed attribute reflects current state

### 2. Behavioral Modes

#### Mode A: Allow Manual Edit OFF (Current Behavior)
- Document textarea becomes `readOnly=true` when corrections exist
- Textarea gets `locked` CSS class to disable mouse interactions
- Global keyboard shortcuts control correction navigation:
  - Arrow Left/Right: Navigate corrections
  - Enter: Accept correction
  - Backspace/Delete: Reject correction
- Any text input triggers `resetState()` clearing all corrections

#### Mode B: Allow Manual Edit ON (New Behavior)
- Document textarea remains editable (`readOnly=false`) during corrections
- No `locked` class applied
- Corrections remain visible as highlights
- Global keyboard shortcuts only work when textarea is NOT focused
- When textarea has focus, normal typing behavior takes precedence
- Text changes update correction positions dynamically or mark conflicts

### 3. State Management

```javascript
// New module-level variables
let allowManualEdit = false; // Loaded from localStorage
let lastTextValue = ''; // For change tracking
```

**Initialization:**
- Read `allowManualEdit` from localStorage on page load
- Set toggle button visual state accordingly
- Apply current mode immediately if corrections exist

**Toggle Behavior:**
- Click flips `allowManualEdit` boolean
- Persist new value to localStorage  
- If corrections currently exist, immediately apply/remove lock based on new mode

### 4. Change Tracking System

When `allowManualEdit` is ON and user types:

1. **Detect Changes**: Compare `documentInput.value` with `lastTextValue`
   - Find first differing character from start (change start position)
   - Find first matching character from end (determines removed/inserted lengths)
   - Create change descriptor: `{start, removedLength, insertedText}`

2. **Update Correction Positions**: For each existing correction:
   - **Before change**: No adjustment needed
   - **After change**: Shift both start/end by net length delta
   - **Overlapping change**: Mark correction as "conflict"

3. **Debounced Redraw**: Use `requestAnimationFrame` to redraw highlights smoothly

### 5. Enhanced Correction Data Model

Extend correction objects with new fields:

```javascript
{
  // Existing fields
  original: string,
  corrected: string,
  explanation: string,
  position: {start: number, end: number},
  
  // New fields (minimal for MVP)
  state: 'active' | 'conflict'   // Default: 'active'
}
```

### 6. Correction States

- **Active**: Original correction, position accurate, can be accepted/rejected
- **Conflict**: Position ambiguous due to overlapping edits, requires manual resolution


### 7. Keyboard Handling Updates

**Current global keydown handler modifications:**

```javascript
document.addEventListener('keydown', (e) => {
  // NEW: In manual edit mode, let textarea handle its own input
  if (allowManualEdit && document.activeElement === documentInput) {
    return; // Let normal typing happen
  }
  
  // Existing correction navigation logic...
  if (e.key === 'ArrowRight') {
    // ... navigate corrections
  }
  // etc.
});
```

**Additional alternate shortcuts** (always work regardless of focus):
- Alt + Left/Right: Navigate corrections
- Ctrl/Cmd + Enter: Accept correction  
- Ctrl/Cmd + Backspace/Delete: Reject correction

### 8. Visual Indicators

**Correction State Styling:**
- **Active**: Current highlight style (yellow background)
- **Conflict**: Muted warning color, disabled cursor

**Popover Updates:**
- Show state badge ("Conflict") near explanation
- Disable Accept button for conflict state
- For conflicts: Show original suggestion as reference

**Mode Indicator:**
Small text near navigation: "Manual editing enabled" when toggle is ON

### 9. Accept/Reject Validation

**Before Accept:**
1. If state is "conflict": Block with guidance message
2. If state is "active": Validate current text at position matches `original`

**Accept/Reject Behavior:**
- Accept: Apply change, shift downstream positions (existing logic)
- Reject: Remove correction (existing logic)
- Undo: Unchanged - only covers Accept/Reject actions, not manual typing

### 10. Conflict Resolution

**Manual Resolution:**
- User must manually fix conflicts by:
  - Accepting/rejecting before the conflicting area
  - Or clearing all corrections and re-analyzing

### 11. Testing & Validation

**Functional Tests:**
- Toggle OFF preserves all current behavior
- Toggle ON allows normal typing during corrections
- Position updates work correctly for before/after edits
- Overlapping edits properly mark conflicts
- Keyboard shortcuts work in both modes

**Basic Edge Cases:**
- Large paste operations affecting multiple corrections
- Rapid typing maintaining smooth performance

**Accessibility:**
- Toggle button keyboard accessible
- ARIA states properly announced

### 12. Implementation Priority

**Phase 1 (MVP - Core Manual Edit Mode):**
1. Add toggle checkbox in correction controls section
2. Implement mode switching (lock/unlock behavior)
3. Basic change tracking and position updates
4. Simple conflict detection
5. Alternate keyboard shortcuts
6. Conflict state visual indicators

**Phase 2 (Polish):**
7. Comprehensive testing
8. Documentation updates
9. Performance optimizations

### 13. Backwards Compatibility

- Default toggle state is OFF - no behavior change for existing users
- All existing keyboard shortcuts and workflows remain unchanged
- New features are purely additive
- Can be disabled entirely by removing toggle if needed

### 14. Success Metrics

- Users can type naturally while corrections are visible
- Basic position tracking works for common edits (typing, backspace, simple paste)
- No performance degradation during normal typing
- Zero regressions in existing correction workflow

---

## Future Enhancements (Post-MVP)

These features are not part of the core manual edit mode but could enhance the overall experience:

### Advanced Position Tracking
- **Smart Re-anchoring**: Search for moved text within a window around conflicts
- **Stale State**: Re-anchored corrections marked as "stale" but still usable  
- **Context Anchors**: Store surrounding text for better re-matching
- **Unique IDs**: Stable correction identifiers for tracking
- **Performance Optimizations**: 
  - Viewport-based updates (only process visible corrections)
  - Batch processing with `requestIdleCallback`
  - Input event optimization using `inputType` and `data` properties

### Enhanced UX Features
- **Skip Functionality**: "Skip this correction" button with dedicated state
- **Auto-dismiss**: Conflicts disappear after N characters typed past them
- **Navigation Enhancements**: Show state counts ("2 active • 3 conflict")
- **Targeted Re-analysis**: Re-analyze small windows around conflicts

### Scalability & Performance  
- **Virtual Scrolling**: For documents with 100+ corrections
- **Web Workers**: Offload heavy position calculations
- **Memory Management**: Limit correction cache and undo history
- **Configurable Thresholds**: User-adjustable re-anchor windows, auto-dismiss timing

### Advanced Edge Cases
- Duplicate text snippets in document (proper unique matching)
- LaTeX with complex escaping
- Multi-cursor/multi-selection editing support
- Undo/redo interaction with position tracking  
- Find & replace operations
- External paste with formatting preservation

### Analytics & Optimization
- Usage telemetry for feature adoption
- Performance metrics collection
- Auto-tuning based on user success rates
- Conflict pattern analysis for better algorithms
