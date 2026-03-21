# CLAUDE.md

We are building the app described in @SPEC.md.

SPEC.md is the authoritative source for:

- system architecture
- database schema
- data flow
- constraints and invariants

Only refer to @SPEC.md when:

- making architectural decisions
- modifying database schema or queries
- implementing features that depend on data flow or auth rules
- behavior is unclear or ambiguous

Do not load it unnecessarily for small, isolated changes.

Keep responses concise and focused. Avoid verbosity, filler, and unnecessary explanations. Provide only essential code, using minimal, targeted snippets.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bun run dev      # Start development server
bun run build    # Production build
bun run start    # Start production server
bun run lint     # Run ESLint
```

All commands use **Bun** as the runtime and package manager (not npm/yarn/pnpm).

## Architecture

This is a **personal note-taking web app** built with Next.js App Router. See `SPEC.md` for the full technical specification.

### Data flow

- **Authenticated routes**: Client Components → Server Actions → SQLite (via Bun's native SQLite client with raw SQL)
- **Public share routes**: Route Handler → SQLite → rendered page

### Key architectural decisions

- **Server Actions** handle all CRUD operations (no REST API for authenticated operations)
- **TipTap** editor stores content as **ProseMirror JSON** in `notes.content_json` — no conversion layer
- **better-auth** manages the `user`, `session`, `account`, and `verification` tables — never modify these directly
- **SQLite** database lives at `/data/app.db`
- Public note sharing uses random UUID tokens stored in `notes.share_token`

### Planned directory structure (from SPEC)

```
app/
  notes/           # Dashboard, create, view, edit pages
  share/[token]/   # Public read-only note view
  api/share/[token]/  # Route handler for public share resolution

lib/
  db/database.ts   # Bun SQLite client singleton
  auth/auth.ts     # better-auth config
  notes/queries.ts # Raw SQL queries
  notes/actions.ts # Server Actions

components/
  editor/          # TipTap note-editor, read-only-editor, extensions
  ui/
```

### Database schema

Auth tables are managed by better-auth. The only app-owned table:

```sql
CREATE TABLE notes (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  title TEXT NOT NULL,
  content_json TEXT NOT NULL,   -- ProseMirror JSON
  share_token TEXT UNIQUE,      -- NULL means not shared
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES user(id)
);
```

### TipTap extensions

`StarterKit`, `Heading`, `Bold`, `Italic`, `Code`, `CodeBlock`, `BulletList`, `ListItem`, `HorizontalRule`. Markdown shortcuts work automatically via StarterKit.

### Authorization

All note CRUD operations must verify server-side that the requesting user owns the note. Public `/share/[token]` pages require no auth.
