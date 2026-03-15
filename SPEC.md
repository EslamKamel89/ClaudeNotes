# Technical Specification

## Project: Note Taking Web App

---

# 1. System Overview

## What the Application Is

A **personal note-taking web application** where authenticated users can create and manage rich text notes using a TipTap editor.

Each note contains:

- a **title**
- **rich text content**

Users can optionally generate a **public share link** allowing others to view a note in read-only mode.

---

## Core Features

Authenticated users can:

- create notes
- view their notes
- edit notes
- delete notes
- share notes publicly
- revoke public sharing

Public users can:

- view shared notes via a **share link**

---

# 2. Technology Stack

## Runtime

```
Bun
```

Used for:

- running Next.js
- SQLite client
- development server

---

## Framework

```
Next.js (App Router)
```

Architecture features used:

- Server Components
- Server Actions
- Route Handlers

---

## Language

```
TypeScript
```

---

## Styling

```
TailwindCSS
```

---

## Authentication

```
better-auth
```

Responsibilities:

- user registration
- login
- session management
- authentication verification

---

## Rich Text Editor

```
TipTap
```

Content format:

```
ProseMirror JSON
```

Stored **exactly as produced by TipTap**.

---

## Database

```
SQLite
```

Accessed via:

```
Bun SQLite client
```

Queries will use:

```
Raw SQL
```

---

# 3. High-Level Architecture

The application uses a **modern Next.js architecture**.

```
Browser
   │
   ▼
React UI (Client Components)
   │
   ▼
Server Components
   │
   ▼
Server Actions
   │
   ▼
SQLite Database
```

For public pages:

```
Public Request
   │
   ▼
Route Handler
   │
   ▼
Database
```

---

# 4. Application Structure

Recommended directory structure:

```
app
 ├ layout.tsx
 ├ page.tsx

 ├ notes
 │   ├ page.tsx
 │   ├ new
 │   │   └ page.tsx
 │   └ [noteId]
 │       ├ page.tsx
 │       └ edit
 │           └ page.tsx

 ├ share
 │   └ [shareToken]
 │       └ page.tsx

 └ api
     └ share
         └ [shareToken]
             └ route.ts
```

Supporting directories:

```
lib
 ├ db
 │   └ database.ts
 │
 ├ auth
 │   └ auth.ts
 │
 ├ notes
 │   ├ queries.ts
 │   └ actions.ts
 │
 └ utils
```

Editor components:

```
components
 ├ editor
 │   ├ note-editor.tsx
 │   ├ read-only-editor.tsx
 │   └ extensions.ts
 │
 └ ui
```

---

# 5. Database Design

Authentication tables are fully managed by **better-auth**. Do not manually modify these tables — interact with them only through better-auth.

## Auth Tables (managed by better-auth)

### user

```sql
CREATE TABLE user (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT NOT NULL UNIQUE,
  email_verified BOOLEAN NOT NULL DEFAULT 0,
  image TEXT,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL
);
```

### session

```sql
CREATE TABLE session (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  token TEXT NOT NULL UNIQUE,
  expires_at DATETIME NOT NULL,
  ip_address TEXT,
  user_agent TEXT,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  FOREIGN KEY (user_id) REFERENCES user(id)
);
```

### account

```sql
CREATE TABLE account (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  account_id TEXT NOT NULL,
  provider_id TEXT NOT NULL,
  access_token TEXT,
  refresh_token TEXT,
  access_token_expires_at DATETIME,
  refresh_token_expires_at DATETIME,
  scope TEXT,
  id_token TEXT,
  password TEXT,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,
  FOREIGN KEY (user_id) REFERENCES user(id)
);
```

### verification

```sql
CREATE TABLE verification (
  id TEXT PRIMARY KEY,
  identifier TEXT NOT NULL,
  value TEXT NOT NULL,
  expires_at DATETIME NOT NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL
);
```

---

## Application Tables

### notes

```sql
CREATE TABLE notes (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,

  title TEXT NOT NULL,

  content_json TEXT NOT NULL,

  share_token TEXT UNIQUE,

  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (user_id) REFERENCES user(id)
);
```

---

# 6. Data Model

## Note Interface

```ts
interface Note {
  id: string;
  userId: string;
  title: string;
  contentJson: string;
  shareToken: string | null;
  createdAt: string;
  updatedAt: string;
}
```

---

# 7. TipTap Content Model

Content will be stored as **ProseMirror JSON**.

Example:

```json
{
  "type": "doc",
  "content": [
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [{ "type": "text", "text": "My Note" }]
    }
  ]
}
```

This JSON will be:

- stored directly in `notes.content_json`
- parsed when rendering

No conversion layer is required.

---

# 8. Editor Configuration

TipTap will use the following extensions:

```
StarterKit
Heading
Bold
Italic
Code
CodeBlock
BulletList
ListItem
HorizontalRule
```

Capabilities:

- bold
- italic
- headings (3 levels)
- inline code
- code blocks
- bullet lists
- horizontal rule

Markdown shortcuts will work automatically through StarterKit.

Example:

```
# Heading
**bold**
```

---

# 9. Authentication Model

Authentication handled by **better-auth**.

Authentication responsibilities:

- register
- login
- session cookies
- session validation

All note operations require:

```
authenticated user
```

Public share pages **do not require authentication**.

---

# 10. Authorization Rules

Rules enforced server-side.

| Action           | Allowed            |
| ---------------- | ------------------ |
| Create note      | Authenticated user |
| View note        | Owner              |
| Edit note        | Owner              |
| Delete note      | Owner              |
| Share note       | Owner              |
| Unshare note     | Owner              |
| View public note | Anyone             |

---

# 11. Server Actions

CRUD operations will use **Server Actions**.

Actions required:

```
createNote
updateNote
deleteNote
shareNote
unshareNote
```

Responsibilities:

- validate user session
- validate input
- run SQL queries
- return updated state

---

# 12. Public Sharing System

Sharing uses a **share token model**.

Example:

```
/share/8f3a1bcd
```

---

## Share Token Generation

When sharing a note:

1. Generate random token
2. Store in `share_token`
3. Return share URL

Example token:

```
crypto.randomUUID()
```

---

## Revoking Sharing

Revoking share simply:

```
UPDATE notes
SET share_token = NULL
```

After this:

```
/share/{token}
```

will no longer resolve.

---

# 13. Public Note Rendering

Public notes render using **TipTap read-only editor**.

The editor instance will:

```
editable: false
```

Purpose:

- render formatted content
- prevent modification

---

# 14. Data Flow

## Creating a Note

```
User → New Note Page
        ↓
Client Editor
        ↓
Submit
        ↓
Server Action
        ↓
SQLite Insert
        ↓
Return Note ID
```

---

## Updating a Note

```
User → Edit Page
        ↓
Editor Changes
        ↓
Save
        ↓
Server Action
        ↓
SQLite Update
```

---

## Viewing Public Note

```
Browser Request
   │
   ▼
Route Handler
   │
   ▼
Find note by share_token
   │
   ▼
Return page
```

---

# 15. UI Pages

## Pages Required

### Dashboard

```
/notes
```

Shows:

- user notes
- create note button

---

### Create Note

```
/notes/new
```

Contains:

- title input
- TipTap editor

---

### View Note

```
/notes/[noteId]
```

Shows note content.

---

### Edit Note

```
/notes/[noteId]/edit
```

Editable editor.

---

### Public Note

```
/share/[shareToken]
```

Read-only rendering.

---

# 16. Error Handling

Common errors:

| Case                | Handling |
| ------------------- | -------- |
| Note not found      | 404      |
| Unauthorized access | redirect |
| Invalid share token | 404      |
| Database failure    | 500      |

---

# 17. Security Considerations

Security rules:

- all CRUD operations verify user ownership
- share tokens are random UUIDs
- public notes read-only
- no raw HTML rendering

---

# 18. Development Environment

Run locally using:

```
bun run dev
```

SQLite database file:

```
/data/app.db
```

---

# 19. Logging

For development:

```
console.log
```

For production (future):

- structured logging
- error monitoring

---

# 20. Implementation Roadmap

Recommended build order.

---

## Step 1

Initialize project

```
bun create next-app
```

Add:

```
Tailwind
TypeScript
```

---

## Step 2

Install dependencies

```
better-auth
tiptap
sqlite
```

---

## Step 3

Setup SQLite database

Create:

```
lib/db/database.ts
```

---

## Step 4

Create database schema

Tables:

```
users
notes
```

---

## Step 5

Integrate authentication

Setup:

```
better-auth
```

Add login + register pages.

---

## Step 6

Implement notes queries

```
createNote
getNotes
getNote
updateNote
deleteNote
```

---

## Step 7

Create TipTap editor component

```
components/editor/note-editor.tsx
```

---

## Step 8

Create server actions

```
createNote
updateNote
deleteNote
shareNote
unshareNote
```

---

## Step 9

Build UI pages

```
/notes
/notes/new
/notes/[id]
/notes/[id]/edit
```

---

## Step 10

Add public share page

```
/share/[token]
```

---

# Final System Summary

This application is a **modern full-stack Next.js system** using:

- Server Components
- Server Actions
- SQLite
- TipTap JSON storage

The architecture ensures:

- minimal API surface
- secure server-side operations
- simple data model
- extensibility for future features.
