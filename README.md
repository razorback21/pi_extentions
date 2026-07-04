# Pi Todos Extension

A file-based todo management extension for [Pi coding agent](https://pi.ai). Stores todos as individual markdown files under `.pi/todos/`, with JSON frontmatter, session-based locking, subtask support, and a full TUI for interactive management.

## Overview

This extension gives Pi a persistent todo system that survives across sessions. Every todo is a plain markdown file — zero databases, zero external services. You can create, assign, refine, close, and track todos entirely through natural conversation with Pi, or use the `/todos` command for a visual interface.

The extension registers a `todo` tool that Pi's LLM calls automatically whenever it needs to create or manage tasks. You don't need to remember commands — just ask Pi to track something.

## Features

- **🗂️ File-based persistence** — Each todo is a `.md` file. Portable, inspectable, grepable.
- **📋 Full CRUD** — Create, read, update, delete todos through conversation or `/todos`.
- **🔒 Session locking** — Lock files prevent concurrent edits from multiple Pi sessions (TTL + stale lock recovery).
- **👤 Assignment tracking** — Claim a todo so Pi knows you're working on it. Release when done.
- **📦 Subtasks** — Break work into `SUB-<hex>` subtasks with independent status tracking (`open` → `in-progress` → `done`).
- **🏷️ Tags & Filtering** — Tag todos for organization. Fuzzy-search through the `/todos` TUI.
- **⏱️ Priority & Due dates** — Optional `low`/`medium`/`high` priority and ISO due dates.
- **🧹 Auto garbage collection** — Configurable cleanup of closed todos older than N days.
- **🖥️ TUI overlays** — `/todos` opens an interactive selector with search, action menus, and detail view.
- **📊 Structured output** — The `todo` tool returns JSON so Pi's LLM can reason about task state.

## Installation

The extension lives in the `todos/` directory of this repo. Pi auto-discovers extensions from `~/.pi/agent/extensions/*/index.ts`. Install it:

```bash
# Create a symlink so Pi finds the extension globally
ln -sf "$(pwd)/todos" ~/.pi/agent/extensions/todos
```

Or copy the directory:

```bash
cp -r todos ~/.pi/agent/extensions/todos
```

Then in Pi, run `/reload` to pick it up. You should see the extension load without errors.

## Usage

### Via conversation (recommended)

Just talk to Pi naturally:

> _"Create a todo to refactor the auth module"_
> _"What todos do I have?"_
> _"Mark the auth refactor as in-progress"_
> _"Add a subtask to 'update login form' under the auth todo"_
> _"Close the auth refactor todo"_

Pi handles the `todo` tool calls automatically.

### Via the `/todos` command

Open the interactive todo manager:

```
/todos
```

This opens a searchable list of all todos. Use the on-screen key hints to:

| Key | Action |
|-----|--------|
| ↑ / ↓ | Navigate list |
| Enter | Open action menu for selected todo |
| Ctrl+Shift+W | Quick: assign and "work on" the todo |
| Ctrl+Shift+R | Quick: "refine" the todo (prompts you for details) |
| Esc | Close the todo manager |
| Type | Fuzzy-search todos by id, title, tags, status, or assignment |

From the action menu you can:

| Action | What it does |
|--------|-------------|
| **view** | Show full todo body (scrollable detail overlay) |
| **work** | Claim the todo and prompt Pi to start working on it |
| **refine** | Prompt Pi to ask clarifying questions before writing task details |
| **close / reopen** | Toggle todo completion status |
| **release** | Unassign the todo from the current session |
| **copy path** | Copy the absolute file path to clipboard |
| **copy text** | Copy the title + body to clipboard |
| **delete** | Permanently delete the todo (with confirmation) |

### Structured output (for LLM tool use)

The `todo` tool accepts these actions:

- `list` / `list-all` — List todos (assigned, open, closed sections)
- `get` — Get full details of a specific todo
- `create` — Create a new todo
- `update` — Replace title/body/status/tags/priority/due_date
- `append` — Append text to the todo body
- `delete` — Delete a todo
- `claim` — Assign the todo to the current session
- `release` — Unassign the todo
- `add-subtask` — Add one subtask
- `add-subtasks` — Add multiple subtasks at once
- `update-subtask` — Update a subtask's title or status
- `complete-subtask` — Mark a subtask done
- `delete-subtask` — Remove a subtask
- `batch-update-subtasks` — Update multiple subtasks in one call

## File Format

Todos are stored in `.pi/todos/` (configurable via `PI_TODO_PATH` env var). Each todo is a markdown file named `<id>.md`.

```
.pi/todos/
├── a1b2c3d4.md       # Each todo gets a random 4-byte hex id
├── e5f67890.md
├── settings.json      # Global settings (GC config)
└── a1b2c3d4.lock      # Lock file (temporary, created during edits)
```

### Todo file structure

```markdown
{
  "id": "a1b2c3d4",
  "title": "Refactor auth module",
  "tags": ["backend", "auth"],
  "status": "in-progress",
  "created_at": "2026-07-04T10:00:00.000Z",
  "assigned_to_session": "session-abc123.json",
  "subtasks": [
    { "id": "dead", "title": "Update login form", "status": "done", "created_at": "..." },
    { "id": "beef", "title": "Add tests", "status": "open", "created_at": "..." }
  ],
  "priority": "high",
  "due_date": "2026-07-11"
}

Detailed notes about the auth refactor go here.
Include any context, acceptance criteria, or links.
```

### Settings

`.pi/todos/settings.json`:

```json
{
  "gc": true,
  "gcDays": 7
}
```

- `gc` — Auto-delete closed todos older than `gcDays` (runs on startup)
- `gcDays` — Age threshold in days (default: 7)

## Requirements

- [Pi coding agent](https://pi.ai) with extension support
- Node.js 18+
- TUI mode for the interactive `/todos` command

## Development

```bash
# Install dependencies
cd todos
npm install

# Symlink for testing
ln -sf "$(pwd)" ~/.pi/agent/extensions/todos
```

## Architecture

```
┌─────────────────────┐
│   Pi LLM            │  ← calls `todo` tool naturally
│   Conversation      │
└──────┬──────────────┘
       │ todo tool calls
       ▼
┌─────────────────────┐
│   Extension (TS)    │  ← todos/todos.ts
│                     │
│  - Tool handlers    │
│  - Lock management  │
│  - File I/O         │
│  - TUI components   │
└──────┬──────────────┘
       │ read/write .md files
       ▼
┌─────────────────────┐
│   .pi/todos/        │  ← File-based storage
│   a1b2.md           │
│   settings.json     │
└─────────────────────┘
```

Key design decisions:

- **Files over DB** — Each todo is a standalone file. No database to install, no schema migrations, trivially grepable.
- **JSON frontmatter** — Structured metadata in JSON (not YAML) for reliable parse/serialize with no dependency.
- **Lock files** — `.lock` files with process PID + session ID prevent race conditions. Stale locks auto-detect after 5min TTL and prompt to steal in interactive mode.
- **Random hex IDs** — 4-byte random hex avoids collisions without a central counter.
- **Subtask tree** — Subtasks render a visual tree (`├── ✅ SUB-beef Add tests [done]`) in both string output and TUI.

## License

MIT
