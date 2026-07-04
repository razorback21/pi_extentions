# todo — File-based todo management for Pi

A lightweight, file-based todo extension for the Pi coding agent. Each todo is a plain markdown file with JSON frontmatter — no database, no external deps. Tasks are created, claimed, tracked, and completed by the LLM automatically, with a full TUI for human oversight.

---

## Overview

The todo extension brings structured task management into Pi agent sessions. Every todo lives as a standalone file under `.pi/todos/` in the project root:

- **Portable** — plain files, trackable in git, readable in any editor
- **Concurrent-safe** — file-level locking prevents conflicts between parallel agent sessions
- **Self-cleaning** — optional garbage collection removes stale closed todos automatically
- **Agent-native** — the `todo` LLM tool is the primary interaction mode; the `/todos` TUI is for humans
- **Subtasks** — hierarchical decomposition with status tracking at every level
- **Session-aware** — tasks can be claimed by a specific agent session with release semantics

### When would you use this?

| Scenario | How it works |
|----------|-------------|
| **Decomposing a complex task** | Agent auto-creates a parent TODO with subtasks, claims it, works through each subtask one at a time updating status as it goes |
| **Parallel agents** | Each agent lists available todos, claims one, works independently — lock system prevents races |
| **Handoff/checkpoint** | Todos persist on disk. Next session lists what's open and picks up where you left off. |
| **Live status board** | Open `/todos watch` in a terminal pane — subtask trees update in real time as agents complete work |
| **Human oversight** | Open `/todos` to browse, search, close, or delete tasks without touching the LLM |

---

## File format

Each todo is stored as `.pi/todos/<8-hex-id>.md`:

```markdown
{
  "id": "deadbeef",
  "title": "Add pagination to results view",
  "tags": ["frontend", "feature"],
  "status": "open",
  "created_at": "2026-07-03T14:30:00.000Z",
  "assigned_to_session": "sessions/1753029128379.json",
  "subtasks": [
    {
      "id": "a1b2",
      "title": "Design pagination component API",
      "status": "done",
      "created_at": "2026-07-03T14:30:00.000Z",
      "completed_at": "2026-07-03T15:00:00.000Z"
    },
    {
      "id": "c3d4",
      "title": "Implement cursor-based pagination",
      "status": "in-progress",
      "created_at": "2026-07-03T14:30:00.000Z"
    },
    {
      "id": "e5f6",
      "title": "Add integration tests",
      "status": "open",
      "created_at": "2026-07-03T14:30:00.000Z"
    }
  ],
  "priority": "high",
  "due_date": "2026-07-10"
}

- Use cursor-based pagination for the API (not offset-based)
- Return `next_cursor` and `prev_cursor` in the response envelope
- Default page size: 25, max: 100
```

The JSON block is the **frontmatter** — parsed separately from the **body** (markdown notes below). The two regions are joined by a blank line. Editing the file directly is fully supported; the extension re-reads it on every access.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                   Pi Agent Session                            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   LLM Tool API                    /todos Command (TUI)       │
│   ───────────────                ─────────────────           │
│   todo(action, id, ...)          Main list + actions          │
│   ↓ structured JSON              Detail overlay (markdown)    │
│   ↓ auto-decompose subtasks      Action menu (8 entries)      │
│   ↓ auto-claim + progress        Delete confirm dialog        │
│                                  Watch dashboard (live)       │
│                                                              │
├──────────┬──────────┬──────────────┬────────────────────────┤
│  Lock    │  Read    │   Write      │  GC Runner             │
│  System  │  (parse) │  (serialize) │  (session_start hook)  │
├──────────┴──────────┴──────────────┴────────────────────────┤
│                                                              │
│                   .pi/todos/  (filesystem)                   │
│                                                              │
│   deadbeef.md   cafebabe.md   settings.json   *.lock         │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Components

| Component | File | Lines | Role |
|-----------|------|-------|------|
| **Storage** | `*.md` in `.pi/todos/` | — | Each file = one todo. JSON frontmatter + markdown body. |
| **Lock System** | `*.lock` files | 100+ | Per-todo exclusive lock with PID, session ID, timestamp. 5-min TTL. Exponential backoff (6 retries). Interactive stale-steal prompt. |
| **Garbage Collector** | `garbageCollectTodos()` | 40+ | Runs on `session_start`. Deletes closed todos older than `gcDays` (default 7). Configurable via `settings.json`. |
| **LLM Tool** | `pi.registerTool({ name: "todo", ... })` | 500+ | 15+ parameterized actions. Prompt snippet + guidelines auto-tell the agent to use subtasks. |
| **Selector** | `TodoSelectorComponent` | 130+ | Searchable list. Fuzzy match on id/title/tags/status/assignment. 10-item visible window with scroll indicators. |
| **Action Menu** | `TodoActionMenuComponent` | 50+ | Context-sensitive actions (view, work, refine, close/reopen, release, copy path/text, delete). |
| **Detail Overlay** | `TodoDetailOverlayComponent` | 100+ | Centered overlay. Markdown rendering. Scrollable (↑↓←→/PgUp/PgDn). Enter = work, Esc = back. |
| **Delete Confirm** | `TodoDeleteConfirmComponent` | 30+ | Yes/No confirmation dialog. |
| **Watch Dashboard** | `/todos watch` inline | 80+ | Live-refreshing pane via `fs.watch`. Subtask trees with emoji icons. Freezes when all done. |

### Data flow

```
Create                         List                           Claim
──────                         ────                           ─────
lock file                      readdir(.pi/todos/)            lock file
  ↓                              ↓                              ↓
generate 8-hex id              filter *.md                    read .md
  ↓                              ↓                              ↓
write .md with JSON FM         parse JSON frontmatter         update assigned_to_session
  ↓                              ↓                              ↓
release lock                   sort + filter (fuzzy)          write .md
                               return list                    release lock

GC on start
────────────
read settings.json
  ↓
scan all .md files
  ↓
parse created_at
  ↓
if status=closed AND age>gcDays → unlink

Subtask lifecycle
─────────────────
add-subtask(id, title)
  →
  random 4-hex id → push → sort open-first → write
update-subtask(id, sub_id, {status, title})
  →
  find → mutate → record completed_at on done → sort → write
```

---

## Agent tool: `todo`

The extension registers a single tool with 15+ action verbs. The LLM is auto-prompted to use it when it encounters multi-step work.

### Prompt guidelines (auto-injected)

Every time the agent loads, these guidelines are added:

> When you encounter a task that benefits from being broken into smaller steps, use the todo tool to create a TODO with subtasks, claim it, and work through each subtask one at a time — updating status as you go.
>
> Use `add-subtask` (or `add-subtasks` for batch) to decompose work, `update-subtask` to mark progress, and `complete-subtask` when a step is done.
>
> The user can run `node /tmp/live-tree.mjs` in another terminal to watch the tree update live.

### Actions reference

#### CRUD

| Action | Required | Optional | Description |
|--------|----------|----------|-------------|
| `create` | `title` | `tags`, `body`, `status`, `priority`, `due_date` | Creates a new todo file. Returns auto-generated `TODO-<8hex>` id. |
| `get` | `id` | — | Full read including body markdown and subtasks. |
| `update` | `id` | `title`, `tags`, `body`, `status` | Replace fields. `body` overwrites (use `append` to add). |
| `append` | `id`, `body` | — | Appends markdown to the body without replacing existing content. |
| `delete` | `id` | — | Permanently removes the `.md` file. Requires lock. |
| `list` | — | — | Returns open todos only (assigned first, then unassigned). |
| `list-all` | — | — | Returns all todos — open, assigned, and closed. |

#### Session assignment

| Action | Required | Optional | Description |
|--------|----------|----------|-------------|
| `claim` | `id` | `force` | Assigns current session to the todo. Without `force`, refuses if another session owns it. |
| `release` | `id` | `force` | Unassigns the current session. Without `force`, only the owning session can release. |

#### Subtasks

| Action | Required | Optional | Description |
|--------|----------|----------|-------------|
| `add-subtask` | `id`, `subtask_title` | — | Adds one subtask. Returns `SUB-<4hex>` id. |
| `add-subtasks` | `id`, `subtask_titles` (array) | — | Batch-add multiple subtasks in one atomic write. |
| `update-subtask` | `id`, `subtask_id` | `subtask_title`, `subtask_status` | Update title and/or status. `completed_at` auto-set on status=`done`, cleared on reopen. |
| `complete-subtask` | `id`, `subtask_id` | — | Shortcut for `update-subtask` with status=`done`. |
| `delete-subtask` | `id`, `subtask_id` | — | Removes the subtask from the array. |
| `batch-update-subtasks` | `id`, `subtask_updates` (array) | — | Multiple subtask updates in one atomic write. Each entry: `{subtask_id, status?, title?}`. |

### Id formats

```
TODO-deadbeef       ← display format (8 hex chars)
deadbeef            ← accepted as input (prefix stripped)
SUB-a1b2            ← subtask display format (4 hex chars)
a1b2                ← accepted as input
```

### Serialization

When the tool returns data to the LLM, todos are serialized as JSON with a rendered subtask tree:

```json
{
  "id": "TODO-deadbeef",
  "title": "Add pagination",
  "tags": ["frontend"],
  "status": "in-progress",
  "created_at": "2026-07-03T14:30:00.000Z",
  "assigned_to_session": "sessions/...",
  "subtasks": [
    { "id": "a1b2", "title": "Design API", "status": "done", "created_at": "...", "completed_at": "..." },
    { "id": "c3d4", "title": "Implement", "status": "in-progress", "created_at": "..." },
    { "id": "e5f6", "title": "Tests", "status": "open", "created_at": "..." }
  ],
  "subtask_tree": "🔶 TODO-deadbeef \"Add pagination\" [in-progress]\n├── ✅ SUB-a1b2 Design API [done] • 2026-07-03T15:00:00.000Z\n├── 🔶 SUB-c3d4 Implement [in-progress]\n└── ⬜ SUB-e5f6 Tests [open]",
  "body": "- Use cursor-based pagination\n- Default page size: 25"
}
```

For list operations, the response is grouped:

```json
{
  "assigned": [ /* todos claimed by the current session */ ],
  "open": [ /* unassigned todos */ ],
  "closed": [ /* done/cancelled todos */ ]
}
```

Each list entry includes a `subtask_summary` when subtasks exist:

```json
{
  "id": "TODO-deadbeef",
  "title": "Add pagination",
  "subtask_summary": { "total": 3, "open": 1, "in_progress": 1, "done": 1 }
}
```

---

## TUI: `/todos` command

Type `/todos` in any Pi chat session to open the interactive todo manager. The TUI is built with Pi's native TUI components (`Container`, `Input`, `SelectList`, `Markdown`, `Text`, `DynamicBorder`).

### Entry point

```
/todos                    ← open with full list (search starts empty)
/todos search pagination  ← open with search pre-filled
/todos watch              ← open live-refreshing dashboard
```

### Main list (selector)

```
┌────────────────────────────────────────────────────────────┐
│  Todos (4 open, 2 closed)                                 │
│                                                            │
│  ▐ search pagination ▌                                     │
│                                                            │
│  → TODO-deadbeef Add pagination      [frontend] (open)     │
│    TODO-cafebabe Refactor auth              (open)         │
│    TODO-b00b1e5 Fix memory leak       [perf] (in-progress) │
│    TODO-12345678 Add integration tests  [qa] (open)        │
│                                                            │
│  Type to search • ↑↓ select • Enter actions               │
│  Ctrl+Shift+W work • Ctrl+Shift+R refine • Esc close       │
└────────────────────────────────────────────────────────────┘
```

Features:
- **Fuzzy search** across id, title, tags, status, and assignment text
- **Pagination** — shows 10 items at a time with scroll offset (`▐ 3/14 ▐`)
- **Quick actions** — `Ctrl+Shift+W` bypasses the action menu and immediately sets up the editor to work
- **Quick refine** — `Ctrl+Shift+R` bypasses the action menu and prompts the agent to ask clarifying questions

Sorting within the list:
1. Open + assigned (current session first)
2. Open + unassigned
3. Closed (only in expanded view or `list-all`)

Within group: oldest `created_at` first.

### Action menu

```
┌─────────────────────────────────────────────────────────┐
│  Actions for TODO-deadbeef "Add pagination"              │
│                                                         │
│  → view       View full todo body                       │
│    work       Exit TUI and start working on this todo   │
│    refine     Exit TUI and refine the task definition   │
│    close      Mark todo as closed                       │
│    release    Release session assignment                │
│    copy path  Copy absolute file path to clipboard      │
│    copy text  Copy title + body as markdown to clipboard│
│    delete     Permanently delete (with confirmation)    │
│                                                         │
│  Enter to confirm • Esc back                            │
└─────────────────────────────────────────────────────────┘
```

Context-sensitive rules:
- If todo is **closed**, the menu shows **reopen** instead of close
- If todo has an **assignment**, the menu shows **release**
- **delete** opens a separate confirm dialog before acting

### Detail overlay

```
┌─────────────────────────────────────────────────────────┐
│ ────────────── Add pagination ──────────────────────────│
│ TODO-deadbeef • in-progress • frontend                  │
│                                                         │
│ - Use cursor-based pagination for the API (not          │
│   offset-based)                                         │
│ - Return next_cursor and prev_cursor in the response    │
│ - Default page size: 25, max: 100                       │
│                                                         │
│ ✅ SUB-a1b2 Design component API                        │
│ 🔶 SUB-c3d4 Implement cursor-based pagination           │
│ ⬜ SUB-e5f6 Write tests                                 │
│                                                         │
│ enter work on todo • esc back • ↑/↓: move. ←/→: page.  │
│ 3-5/12                                                  │
└─────────────────────────────────────────────────────────┘
```

- Full markdown rendering via Pi's `Markdown` component
- Subtask tree rendered directly in the overlay with emoji status icons
- Scrollable: `↑`/`↓` for line, `←`/`→` for page
- `Enter` exits to "work on this todo"
- `Esc` returns to the action menu

### Delete confirm

```
┌─────────────────────────────────────────────────────────┐
│  Delete todo TODO-deadbeef? This cannot be undone.      │
│                                                         │
│  → Yes                                                  │
│    No                                                   │
│                                                         │
│  Enter to confirm • Esc back                            │
└─────────────────────────────────────────────────────────┐
```

### Watch dashboard: `/todos watch`

Opens a live-refreshing read-only dashboard. Unlike the main list, this pane auto-updates via `fs.watch` every time a `.md` file changes.

```
● LIVE  4 active todos · auto-refresh
────────────────────────────────────────────────────────────
⬜ TODO-deadbeef "Add pagination" [frontend]
├── ✅ SUB-a1b2 Design API [done]
├── 🔶 SUB-c3d4 Implement pagination [in-progress]
├── ⬜ SUB-e5f6 Write unit tests [open]
└── ⬜ SUB-789a Add integration tests [open]

🔶 TODO-cafebabe "Refactor auth" (assigned: session-abc, current)
  [perf]

⬜ TODO-b00b1e5 "Fix memory leak"

Esc to close · auto-refreshes on file changes
```

Key behaviors:
- **Only active todos** displayed (assigned + open). Closed/cancelled are hidden.
- **Subtask trees** render with `├──` / `└──` branches and status emoji
- **Session indicator** — `(assigned: session-abc, current)` shows current vs. other session
- **Auto-freeze** — when all active todos are completed, the dashboard freezes with:

```
✅ All todos completed
────────────────────────────────────────────────────────────
  All tasks are done.

  Everything completed!

Esc to close · frozen (no background processing)
```

- **Zero polling** — pure `fs.watch` events. No CPU usage between changes.
- The watcher stops when the dashboard closes — no resource leaks.

### Tool result rendering

When the LLM calls `todo`, the results render inline in the chat:

**List (collapsed):**
```
Assigned todos (1)
  TODO-deadbeef Add pagination [frontend] (assigned: session-abc, current)
Open todos (3)
  TODO-cafebabe Refactor auth (open)
  TODO-b00b1e5 Fix memory leak [perf] (open)
Closed todos (1)
  TODO-ff123456 Old cleanup (closed)

(app.tools.expand to expand)
```

**List (expanded):**
```
Assigned todos (1)
  TODO-deadbeef Add pagination [frontend]
Open todos (3)
  TODO-cafebabe Refactor auth
  TODO-b00b1e5 Fix memory leak [perf]
  TODO-12345678 Add integration tests [qa]
Closed todos (1)
  TODO-ff123456 Old cleanup

Status: in-progress
Tags: frontend
Created: 2026-07-03T14:30:00.000Z

Body:
  - Use cursor-based pagination
  - Return next_cursor + prev_cursor

Subtasks (3):
  ✅ SUB-a1b2 "Design API" [done] • 2026-07-03T15:00:00.000Z
  🔶 SUB-c3d4 "Implement pagination" [in-progress]
  ⬜ SUB-e5f6 "Write unit tests" [open]
```

**Create/Update/Claim (collapsed):**
```
✓ Created TODO-deadbeef Add pagination [frontend] (app.tools.expand to expand)
✓ Claimed TODO-deadbeef Add pagination [frontend]
✓ Completed subtask SUB-c3d4 Implement pagination
```

---

## Configuration

### Settings

Create `.pi/todos/settings.json`:

```json
{
  "gc": true,
  "gcDays": 7
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `gc` | boolean | `true` | Master switch for garbage collection |
| `gcDays` | number | `7` | Age (in days since `created_at`) after which a closed todo is permanently deleted |

GC runs once on session start. To disable entirely: `{ "gc": false }`.

### Environment variable

```bash
export PI_TODO_PATH="/path/to/shared/todos"
```

Overrides the todo directory. When unset, defaults to `.pi/todos/` relative to the project root. Useful for:
- Sharing a todo pool across multiple Pi projects
- Pointing to a git-tracked directory outside `.pi/`
- Centralizing tasks for a monorepo workspace

---

## Lock system

Every write operation acquires an exclusive lock:

```
1.  Open <id>.lock with O_EXCL (fails if file exists)
2.  Write LockInfo { id, pid, session, created_at }
3.  Perform read/write
4.  Unlink .lock file
```

| Aspect | Behavior |
|--------|----------|
| **Retry** | 6 attempts with exponential backoff: 200ms, 400ms, 600ms, 800ms, 1000ms, 1200ms |
| **TTL** | 5 minutes. A lock older than this is considered stale (crashed session). |
| **Stale recovery (interactive)** | User is prompted: *"Todo appears locked (stale). Steal the lock?"* |
| **Stale recovery (non-interactive)** | Error returned: *"Todo lock is stale. Restart pi or run in interactive mode to steal it."* |
| **Busy message** | *"That task is busy syncing changes. Try again in a moment."* |
| **Lock file location** | Same directory as the todo `.md` file, e.g. `.pi/todos/deadbeef.lock` |

---

## Subtask system

Subtasks allow hierarchical decomposition of work.

### Properties

| Field | Type | Description |
|-------|------|-------------|
| `id` | `string` (4-hex) | Auto-generated, e.g. `"a1b2"`. Displayed as `SUB-a1b2`. |
| `title` | `string` | Required. Cannot be empty. |
| `status` | `"open" \| "in-progress" \| "done"` | Lifecycle state. |
| `created_at` | ISO timestamp | Set on creation. |
| `completed_at` | ISO timestamp | Auto-set when status changes to `done`. Auto-cleared when reopened. |

### Sorting

Subtasks are sorted in the file and in all UI surfaces:
1. **Incomplete first** (`open` → `in-progress`)
2. **Complete last** (`done`)
3. Within group: **oldest `created_at` first**

### Emoji status icons

| Status | Icon |
|--------|------|
| `open` | ⬜ |
| `in-progress` | 🔶 |
| `done` | ✅ |

---

## Status & assignment lifecycle

```
                      claim
         ┌──────────────────────────→  open/assigned
         │                                  │
         │                                  │ release
         │                                  ↓
         │                            open/unassigned
         │                                  │
         │                                  │ close
         │                                  ↓
         │                              closed ──→ GC deletes after gcDays
         ↑                                  │
         └───────────── reopen ─────────────┘
```

Subtask lifecycle:

```
open ──→ in-progress ──→ done
  ↑                        │
  └────── reopen ──────────┘
```

When a todo is closed/reopened via the tool or TUI, `assigned_to_session` is automatically cleared when closing and preserved when reopening.

---

## Priority & due dates

Optional metadata, preserved through read/write round-trips:

| Field | Values | Example |
|-------|--------|---------|
| `priority` | `"low"`, `"medium"`, `"high"` | `"priority": "high"` |
| `due_date` | ISO date string | `"due_date": "2026-07-10"` |

Passed as parameters to `create` and optionally updated via `update`. Available in the serialized tool output and stored in the JSON frontmatter.

---

## Tags

Tags are free-form strings passed on creation or update. The TUI search and tool filter support tag-based matching. Tag display:

```
TODO-deadbeef Add pagination [frontend, feature] (open)
```

Common conventions (not enforced by the extension):
- Workflow: `blocked`, `wip`, `review`
- Type: `bug`, `feature`, `docs`, `perf`, `refactor`, `qa`
- Scope: `frontend`, `backend`, `infra`, `ops`

---

## Live tree watcher

The prompt guidelines tell the agent to inform users they can run:

```bash
node /tmp/live-tree.mjs
```

in another terminal to watch the todo tree update live. This companion script (not part of this extension) reads the `.pi/todos/` directory and renders a live-updating tree as the agent creates and completes subtasks.

---

## Development

### File structure

```
todos.ts                  ← Single-file extension (~2800 lines)
.pi/
  todos/                  ← Created automatically
    settings.json         ← Optional (gc config)
    *.md                  ← Todo data files
    *.lock                ← Transient lock files (don't commit)
```

### Key types

```typescript
interface TodoFrontMatter {
  id: string;                    // 8-hex, e.g. "deadbeef"
  title: string;
  tags: string[];
  status: string;                // "open" | "closed" | "in-progress" | "done"
  created_at: string;            // ISO timestamp
  assigned_to_session?: string;  // session file path
  subtasks?: Subtask[];
  priority?: "low" | "medium" | "high";
  due_date?: string;             // ISO date
}

interface TodoRecord extends TodoFrontMatter {
  body: string;                  // Markdown notes below the JSON block
}

interface Subtask {
  id: string;                    // 4-hex, e.g. "a1b2"
  title: string;
  status: "open" | "in-progress" | "done";
  created_at: string;
  completed_at?: string;
}

interface TodoSettings {
  gc: boolean;                   // default: true
  gcDays: number;                // default: 7
}

interface LockInfo {
  id: string;
  pid: number;
  session: string | null;
  created_at: string;
}
```

### Key functions

| Function | Lines | Role |
|----------|-------|------|
| `parseTodoContent()` | ~15 | Splits JSON frontmatter from markdown body. |
| `serializeTodo()` | ~25 | Builds JSON + body string for writing. |
| `splitFrontMatter()` | ~35 | Scans for `{...}` JSON root, finds the closing `}`. |
| `parseFrontMatter()` | ~50 | Parses JSON block into `TodoFrontMatter` with safe defaults. |
| `findJsonObjectEnd()` | ~25 | Low-level brace matcher (handles escaped strings). |
| `withTodoLock()` | ~15 | Generic wrapper: acquire lock → run fn → release. |
| `acquireLock()` | ~70 | `O_EXCL` file open, exponential backoff, stale detection. |
| `listTodos()` | ~40 | `readdir` + parse each `.md` → sort. |
| `listTodosSync()` | ~35 | Synchronous version (used by watch dashboard). |
| `filterTodos()` | ~50 | Fuzzy search against id, title, tags, status, assignment. |
| `sortTodos()` | ~15 | Assigned first → open → closed. |
| `garbageCollectTodos()` | ~35 | Scan → parse age → unlink stale closed todos. |
| `generateTodoId()` | ~10 | `crypto.randomBytes(4).toString("hex")` collision-checked. |
| `renderSubtasksTree()` | ~20 | ASCII tree with emoji icons. |
| `splitTodosByAssignment()` | ~20 | Categorizes into {assigned, open, closed}. |
| `renderTodoList()` | ~40 | Themed text rendering for tool results. |
| `renderTodoDetail()` | ~35 | Full detail rendering with subtask tree. |
| `appendExpandHint()` | ~5 | Adds "(app.tools.expand to expand)" footer. |

### UI component classes

| Class | Parent | Lines | Role |
|-------|--------|-------|------|
| `TodoSelectorComponent` | `Container` | 130 | Search, filter, paginate, select todos |
| `TodoActionMenuComponent` | `Container` | 50 | Context-sensitive action picker |
| `TodoDeleteConfirmComponent` | `Container` | 30 | Yes/No confirm dialog |
| `TodoDetailOverlayComponent` | *(custom)* | 100 | Scrollable markdown overlay |
| `TodoOverlayAction` | *(type)* | — | `back` \| `work` — return from overlay |
| `TodoMenuAction` | *(type)* | — | 8 action variants for the menu |
| `TodoToolDetails` | *(type)* | — | Discriminated union for tool result details |

### TUI rendering pipeline

```
TodoSelectorComponent.render(width)
  → updateHeader()        "Todos (4 open, 2 closed)"
  → applyFilter(query)    fuzzy search
  → updateList()          visible window of 10 items
      ├─ selected: "→ " + accent color
      ├─ assigned: "(assigned: session, current)" green
      └─ tags: [frontend, qa] muted
  → scroll info if total > 10
```

### Tool result rendering pipeline

```
renderResult(result, expanded, theme)
  → if partial: "Processing..." (warning color)
  → if error: "Error: ..." (error color)
  → if list action: renderTodoList()
      ├─ collapsed: max 3 per section + "N more" dim
      └─ expanded: full list with status/tags/assignment
  → if single todo: renderTodoDetail()
      ├─ heading with id + title + tags + assignment
      ├─ Status / Tags / Created metadata
      ├─ Body lines indented
      └─ Subtask tree (emoji + ASCII branches)
  → if create/update/claim: "✓ Created/Updated/Claimed" prefix
  → if collapsed: append "(app.tools.expand to expand)" hint
```

---

## Installation

### Prerequisites

- Pi coding agent installed
- Node.js runtime (the extension imports from `@earendil-works/pi-coding-agent`, `@earendil-works/pi-tui`)

### Setup

1. Place `todos.ts` somewhere in Pi's extension path (workspace or plugin directory).
2. Pi auto-discovers and loads it on session start.
3. `.pi/todos/` is created automatically on first use.

No configuration needed to get started — todos work out of the box.

---

## FAQ

**Q: Can I manually edit todo files?**
Yes. They're plain markdown. The next `list` call re-reads from disk. The watch dashboard updates instantly via `fs.watch`.

**Q: What happens if two agents edit the same todo?**
The lock system prevents concurrent writes. The second agent gets a "busy" error and auto-retries with backoff. After 5 minutes the lock is considered stale and can be stolen.

**Q: Are todos shared across projects?**
By default, todos are scoped to `.pi/todos/` within each project root. Set `PI_TODO_PATH` to an absolute path to share a pool across projects.

**Q: Can I track todos in git?**
Yes. The `.md` files version naturally. Ignore lock files:

```gitignore
.pi/todos/*.lock
```

**Q: How do I delete all todos?**
```bash
rm -rf .pi/todos/
```
Recreated on next session start.

**Q: What does "refine" do exactly?**
It exits the TUI and sets the editor text to:

> let's refine task TODO-<id> "<title>": Ask me for the missing details needed to refine the todo together. Do not rewrite the todo yet and do not make assumptions. Ask clear, concrete questions and wait for my answers before drafting any structured description.

This forces the agent to gather requirements interactively rather than guessing.

**Q: What does "work" do?**
Sets the editor text to `work on todo TODO-<id> "<title>"` — which triggers the agent to claim the todo (if not already claimed) and begin implementing.

**Q: Is the lock system multi-machine safe?**
No. The lock is a local file with `O_EXCL` semantics. For cross-machine coordination, point `PI_TODO_PATH` at a shared filesystem (NFS/SMB) — but `O_EXCL` behavior on network filesystems varies.

**Q: What's the maximum todo count?**
No hard limit. Each todo is a separate file. `readdir` + parse is O(n). For thousands of todos, consider turning GC on to keep the directory lean.

**Q: How does the list filter work?**
Fuzzy matching across a combined search string: `TODO-<id> <id> <title> <tags> <status> assigns:<session>`. Each space-separated token must fuzzy-match independently. Higher scores (more contiguous matches) rank first.

**Q: The detail overlay shows "No details yet" — how do I add details?**
Use the tool's `update` or `append` action with a `body` field, or `Ctrl+Shift+R` to refine, or `work` to start implementing — the agent will populate the body as it works.

**Q: Can I use this outside Pi / without the LLM?**
The file format is plain markdown — any editor works. But the lock system, TUI, and tool registration are Pi-specific extensions APIs.

**Q: How do subtasks interact with GC?**
GC only looks at the parent todo's `status`. A closed todo with incomplete subtasks will still be GC'd after `gcDays`. Subtasks are deleted with their parent.

**Q: Is there a `reopen` tool action?**
There's no dedicated `reopen` action. Use `update` with `status: "open"` to reopen a closed todo. The TUI action menu shows "reopen" for closed todos.
