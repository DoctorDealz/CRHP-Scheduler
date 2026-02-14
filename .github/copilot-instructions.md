# Copilot Instructions: CRHP Schedule Hub

## Project Overview
**CRHP Schedule Hub** is a web-based schedule management application for organizing church retreats. It consists of three independent single-file HTML applications that share data via browser localStorage.

**Key Apps:**
- **crhp-scheduler.html** (~2090 lines): Editor for creating/editing schedules, roster, and checklists
- **crhp-viewer.html** (~1767 lines): Read-only timeline viewer with live "now" line and task status toggles
- **print-service.html** (~377 lines): Print/PDF service that receives data via postMessage and renders printable schedule tables
- **index.html**: Landing page linking to the three apps

---

## Critical Architecture Patterns

### Data Model & Storage
```javascript
// All data persists in localStorage with v50_ prefix:
localStorage.v50_tasks       // Array of task objects
localStorage.v50_roster      // Array of {name, role} objects  
localStorage.v50_rooms       // Array of room name strings
localStorage.v50_checklist   // Array of checklist item objects
localStorage.v50_days        // Current number of days (int)
```

**Master Template** (crhp-viewer.html lines 813-819): Emergency fallback data loaded if storage is missing. This is used for fresh starts and contains example tasks with "Jacob," "Jonathan," and "Thomas" crew.

**Critical Rule**: Always call `saveAll()` after mutating any data structure. Without it, changes won't persist:
```javascript
const saveAll = () => { 
    localStorage.setItem('v50_tasks', JSON.stringify(tasks)); 
    localStorage.setItem('v50_checklist', JSON.stringify(checklist));
    localStorage.setItem('v50_roster', JSON.stringify(roster)); 
    localStorage.setItem('v50_days', totalDays);
    localStorage.setItem('v50_rooms', JSON.stringify(rooms));
};
```

### Two-Lane Timeline System
Tasks are rendered on **two independent horizontal lanes**: "main" and "bg" (background). Each lane:
- Spans 350px width (fixed layout, not responsive)
- Starts at left: 65px (main) or 495px (bg)
- Has its own header with color: cyan (`--accent`) for main, purple (`--bg-task`) for bg
- Handles overlapping tasks by subdividing column widths

**Positioning formula** (crhp-scheduler.html line 1185):
```javascript
left: (lt==='main' ? 65 : 495) + (colIndex * laneWidth)px;
top: (startMinutes - dayStartMinutes) * 3 + 35px;
height: (endMinutes - startMinutes) * 3px;
```

### Task Status System (Important!)
Tasks have **independent** P (Pending) and C (Complete) flags:
- Toggling P clears C, toggling C clears P ([crhp-viewer.html](crhp-viewer.html#L875-L878))
- Status indicators appear as tiny toggle buttons in task nodes—only if max simultaneous tasks < 3
- Checklist items use identical logic: `item.pending` / `item.done`

### Name Encoding & Roster Lookups
People are stored in tasks as encoded strings: `"Name (Role)"`. When displaying, this splits on `' ('`:
```javascript
const namePart = "Jacob (Facilities leader)".split(' (')[0]  // "Jacob"
const rolePart = "Jacob (Facilities leader)".split(' (')[1].replace(')', '')  // "Facilities leader"
```

When editing roster and person names change, all task references must be **renamed in place** (crhp-scheduler.html ~lines 1440–1460).

### Room Assignment
- **Legacy**: Single `task.room` string (e.g., "Chapel")
- **Current**: Prefer `task.rooms` array for multiple rooms (crhp-scheduler.html line 1203)
- Display logic checks `t.rooms && t.rooms.length > 0` first, falls back to `t.room`

---

## Component-Specific Workflows

### Scheduler (Editor)
**Key Functions:**
- `openModal(taskIndex, isNew)`: Opens task editor modal
- `toggleStatus(index, type, event)`: Toggles P/C flags, calls `saveAll()` and `renderTasks()`
- `openRosterModal(index)`: Edit/delete crew members
- `addTask()`, `deleteTask(index)`: Create/remove tasks with modal dialogs
- `activateDay(day)`, `activateChecklist()`: Switch timeline/checklist view

**Modal Editing Workflow:**
1. Modal opens → reads current task values into input fields
2. User modifies values (name, time, crew, room, description)
3. Modal save → updates task object → `saveAll()` → `renderTasks()`
4. Modal close → resets form state

**Crew Assignment in Task Modal:**
- `renderEditRoster(assigned)` (line 1586) generates checklist of all roster members
- `assigned` is an array of strings: `["Jacob (Facilities leader)", "Thomas (Formation leader)"]`
- Updates stored as `t.people = [...]` array of encoded names

### Viewer (Timeline Display)
**Key Functions:**
- `renderTasks()`: Renders all day's tasks into absolutely-positioned boxes on two lanes
- `updateNowLine()`: Timer updates the "now" red line every tick (shown as current time)
- `renderDays()`: Renders day tabs at top
- `renderChecklist()`: Renders global checklist items
- `toggleStatus()`: Same as scheduler—updates flag, saves, rerenders

**Layout Rendering** (crhp-viewer.html ~lines 1150–1300):
1. Filter tasks by `currentDay` and `type` ("main" or "bg")
2. Calculate day's time range: `dStart` (rounded to nearest 15-min) to `dEnd`
3. Draw hour lines and labels based on 15-minute grid (1-minute = 3px)
4. For each task, calculate overlapping tasks to determine column count
5. Position task node: `left = lane_base + colIndex * laneWidth`, `top = (minutes - dStart) * 3 + 35`

**Time Format Conversions:**
- Internal storage: `"HH:MM"` (24-hour string, e.g., "09:00", "22:30")
- `getMins(timeStr)` converts to minutes since midnight
- `formatTo12h(timeStr)` converts to 12-hour display with AM/PM

### Print Service
**Data Flow:**
1. Receives data via `window.postMessage({type: 'PRINT_DATA', payload: {...}})`
2. Also accepts manual JSON upload (user clicks "UPLOAD JSON" button)
3. Renders printable HTML table for each day, one per page
4. Table layout: Time column + Main lane columns + BG lane columns

**Throbber Logic**: Delayed spinner shown for 200+ ms to avoid flicker. Data loaded into localStorage key `crhp_print_exchange` for persistence across reloads.

---

## Key Implementation Details

### Modal System
- Overlay modal ID: `#taskModal`, `#rosterEditModal`, `#helpModal`
- Close by clicking outside modal content, or X button
- Modal content scrolls vertically if overflow (crhp-viewer.html lines 231–237)
- Re-render needed after any state change affecting displayed data

### Day Management
- Days are **1-indexed** (Day 1, Day 2, etc.)
- Adding a day: increment `totalDays`, `saveAll()`
- Switching days: set `currentDay`, `updateViewToggles()`, `renderDays()`, `renderTasks()`
- No task splitting: tasks belong to a specific `day` number; creating new days doesn't auto-copy

### Timezone & "Now" Line
- Uses local browser time: `new Date()` (no timezone offset handling)
- "Now" line only shows in timeline view, not checklist mode
- Position: `minutes_now = (hour * 60) + minute`, then `pixelPos = (minutes_now - dStart) * 3 + 35`

### Styling & Theming
- CSS variables in `:root`: `--bg`, `--card`, `--accent`, `--bg-task`, etc.
- Scheduler/Viewer use dark theme (#0f172a bg, #38bdf8 cyan accent)
- Print Service uses light theme (#f0f9ff main, #fdf4ff bg lane)
- Selector dropdown custom styled with SVG arrow (crhp-viewer.html lines 177–191)

---

## Common Editing Tasks

### Adding a New Task Field
1. Add input in task modal HTML
2. Read value in `openModal()` before showing
3. Save to `task.fieldName` in save handler
4. Call `saveAll()` and `renderTasks()`

### Changing Timeline Width
- Central constant: **890px** (crhp-scheduler.html line 47, crhp-viewer.html line 112)
- Recalculate gutter positions: left 60px, main lane 350px (to 410px), center gap 80px, bg lane 350px (to 860px)
- Vertical line markers (`.v-line`) use `left: Xpx` positions matching these boundaries

### Modifying Roster Deletion
- Delete row in `roster` array
- Call `removePersonFromTasks(keyName)` to purge from all tasks
- Update `renderEditRoster()` state if modal is open
- `saveAll()` + `renderRoster()` + `renderTasks()`

---

## Testing & Debugging

### Inspecting State
- In console: `JSON.parse(localStorage.v50_tasks)` to view all tasks
- Check `tasks[0]` structure: `{day, start, end, name, desc, people, type, complete, pending, room, rooms}`

### Clearing Data for Fresh Start
- `localStorage.clear()` or selective: `localStorage.removeItem('v50_tasks')`
- Reload page—Master Template will load automatically

### Print Service Testing
- Open Scheduler → confirm data saves
- Open Print Service in new tab
- Click "UPLOAD JSON" → select exported JSON file
- Verify table renders with correct day/task data

---

## File Navigation Quick Reference
| File | Role | Lines | Key Functions |
|------|------|-------|---|
| [crhp-scheduler.html](crhp-scheduler.html) | Editor | 2090 | `openModal()`, `renderTasks()`, `addTask()`, `openRosterModal()` |
| [crhp-viewer.html](crhp-viewer.html) | Viewer | 1767 | `updateNowLine()`, `renderTasks()`, `toggleStatus()`, `renderChecklist()` |
| [print-service.html](print-service.html) | Print | 377 | `render()`, `manualLoad()`, postMessage listener |
| [index.html](index.html) | Landing | 112 | Navigation links to apps |
