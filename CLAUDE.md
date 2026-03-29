# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Next.js + Turbopack)
npm run dev

# Build & start production
npm run build && npm run start

# Run tests
npm test

# Lint
npm run lint

# Database reset (wipes all data)
npm run db:reset
```

All dev/build/start commands include `NODE_OPTIONS='--require ./node-compat.cjs'` — this is intentional for Node.js compatibility.

## Code Style

- Use comments sparingly. Only comment complex or non-obvious code.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates them via a **virtual file system** (no disk I/O) with real-time preview.

### Data Flow

1. User types in `ChatInterface` → sends message to `/api/chat` with current VirtualFileSystem state
2. `/api/chat/route.ts` streams Claude responses using Vercel AI SDK; Claude calls tools (`str_replace_editor`, `file_manager`) to write/modify files
3. Tool results update `FileSystemContext` (in-memory VirtualFileSystem)
4. `PreviewFrame` renders the generated component live
5. On generation complete, project state is persisted to SQLite via Prisma

### Key Abstractions

- **`src/lib/file-system.ts`** — `VirtualFileSystem` class: all file operations happen in memory, never on disk
- **`src/lib/provider.ts`** — LLM provider abstraction; uses real Claude if `ANTHROPIC_API_KEY` is set, falls back to `MockLanguageModel`
- **`src/lib/tools/`** — AI tool definitions (`str-replace.ts`, `file-manager.ts`) that Claude calls to manipulate the virtual FS
- **`src/lib/contexts/`** — React Contexts for `FileSystemContext` and `ChatContext` that coordinate state across the UI
- **`src/actions/`** — Next.js Server Actions for auth and project CRUD (bypass REST, call Prisma directly)

### Auth

Session-based auth using JWT (jose) + bcrypt. `src/middleware.ts` protects `/api/projects` and `/api/filesystem`. Anonymous users can generate components without persistence.

### Database

SQLite via Prisma. Schema at `prisma/schema.prisma`, client generated to `src/generated/prisma`. Models: `User`, `Project`.

### UI Layout

`src/app/main-content.tsx` orchestrates the split-panel layout:
- Left panel: `ChatInterface`
- Right panel: tabs switching between `PreviewFrame` (live render) and `CodeEditor` + `FileTree`

UI components are shadcn/ui (Radix UI + Tailwind CSS v4) in `src/components/ui/`.

### Path Alias

`@/*` maps to `src/*`.
