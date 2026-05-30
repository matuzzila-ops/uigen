# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**UIGen** — an AI-powered React component generator with live preview. Users describe a component in chat; Claude generates JSX files in a virtual file system; the result renders live in a sandboxed iframe.

## Commands

```bash
npm run setup          # first-time: install deps + prisma generate + migrate
npm run dev            # start dev server (Turbopack) at localhost:3000
npm run build          # production build
npm run lint           # ESLint
npm test               # Vitest (watch mode)
npx vitest run         # Vitest single run
npx vitest run src/lib/__tests__/file-system.test.ts  # single test file
npm run db:reset       # drop and recreate SQLite database
```

> **Do not run `npm audit fix`.** Dependencies are pinned to specific compatible versions; audit fix can break the app.

## Architecture

### Request flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Server reconstructs a `VirtualFileSystem` from the serialized file data sent by the client
3. `streamText` (Vercel AI SDK) streams a response from Claude with two tools available: `str_replace_editor` and `file_manager`
4. The client (`ChatContext`) receives the stream and dispatches tool calls to `FileSystemContext.handleToolCall`
5. `FileSystemContext` mutates the in-memory `VirtualFileSystem` and increments `refreshTrigger`
6. `PreviewFrame` reacts to `refreshTrigger`, transforms all files via `createImportMap`, and sets `iframe.srcdoc` with a fresh HTML page

### Virtual file system (`src/lib/file-system.ts`)

All generated code lives in an in-memory `VirtualFileSystem` — nothing is written to disk. The VFS is serialized to JSON and sent with every chat request so the server can reconstruct it. Projects for authenticated users are persisted in SQLite as JSON blobs (`messages` + `data` columns on `Project`).

### Live preview pipeline (`src/lib/transform/`)

- Babel Standalone transforms JSX/TSX files in the browser
- Each file becomes a blob URL
- All local files are wired together via a native browser import map
- Third-party packages (e.g. `lucide-react`) are resolved to `https://esm.sh/<package>`
- Tailwind CSS is loaded from CDN inside the iframe
- The entry point must be `/App.jsx` (or `App.tsx`, `index.jsx`, etc.)

### AI tools

- `str_replace_editor` — create/str_replace/insert operations on VFS files (built in `src/lib/tools/str-replace.ts`)
- `file_manager` — rename and delete VFS files (built in `src/lib/tools/file-manager.ts`)

The system prompt (`src/lib/prompts/generation.tsx`) instructs Claude to always start with `/App.jsx` as the entry point and import local files with the `@/` alias.

### Provider abstraction (`src/lib/provider.ts`)

`getLanguageModel()` returns the real `claude-haiku-4-5` model when `ANTHROPIC_API_KEY` is set to a real key, or a `MockLanguageModel` that returns canned components otherwise. The mock simulates multi-step tool-call sequences so the UI still demonstrates the full flow without an API key.

### Auth (`src/lib/auth.ts` + `src/middleware.ts`)

JWT sessions stored in an `auth-token` HttpOnly cookie (7-day TTL, signed with `JWT_SECRET`). Passwords are hashed with bcrypt. Middleware protects `/api/projects` and `/api/filesystem`. The `/api/chat` route is unprotected but only saves project data when a valid session exists.

### Anonymous work tracking (`src/lib/anon-work-tracker.ts`)

When an anonymous user generates content, the VFS snapshot and chat messages are stored in `sessionStorage`. If the user then signs up, the app offers to migrate that work into their new account.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models: `User` (email + bcrypt password) and `Project` (stores `messages` and `data` as JSON strings). Prisma client is generated to `src/generated/prisma/`.

### Testing

Vitest with jsdom + React Testing Library. Tests live alongside their subjects in `__tests__/` subdirectories. No mocking of the `VirtualFileSystem` — tests instantiate it directly.
