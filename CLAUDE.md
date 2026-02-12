# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, Claude generates the code, and a live preview renders it in an iframe — all within a split-pane IDE-like interface.

## Commands

```bash
npm run setup          # Install deps + generate Prisma client + run migrations
npm run dev            # Start dev server (Next.js + Turbopack)
npm run build          # Production build
npm run lint           # ESLint
npm test               # Run all tests (Vitest)
npx vitest run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run db:reset       # Reset SQLite database
npx prisma migrate dev # Run pending migrations
npx prisma generate    # Regenerate Prisma client after schema changes
```

## Tech Stack

- **Next.js 15** (App Router) / **React 19** / **TypeScript**
- **Tailwind CSS v4** for styling
- **Prisma** with SQLite (`prisma/dev.db`)
- **Vercel AI SDK** (`ai` + `@ai-sdk/anthropic`) for streaming Claude responses
- **Vitest** + React Testing Library for tests
- **Monaco Editor** for code editing, **Babel Standalone** for client-side JSX transformation
- UI primitives from **Radix UI** via shadcn/ui (`src/components/ui/`)

## Architecture

### Request Flow

1. User types a prompt in `ChatInterface`
2. `ChatProvider` (React Context) sends it to `POST /api/chat` via Vercel AI SDK's `useChat`
3. The API route reconstructs a `VirtualFileSystem` from the request body, attaches the system prompt and two AI tools (`str_replace_editor`, `file_manager`), then calls `streamText` with Claude (or a `MockLanguageModel` if no API key)
4. Streamed tool calls arrive on the client; `FileSystemContext` applies them to the in-memory virtual filesystem
5. `PreviewFrame` transforms JSX→JS with Babel Standalone and renders the result in an isolated iframe
6. On completion, authenticated users' projects are persisted to SQLite (messages + file system as JSON)

### Key Abstractions

- **Virtual File System** (`src/lib/file-system.ts`) — in-memory file tree with CRUD ops, serialized as JSON for storage and sent with every chat request. All paths are `/`-prefixed (e.g. `/App.jsx`, `/components/Button.jsx`).
- **AI Tools** (`src/lib/tools/`) — `str_replace_editor` for create/view/str_replace operations, `file_manager` for list/rename/delete. Both operate on the VirtualFileSystem instance.
- **Provider** (`src/lib/provider.ts`) — returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel` that returns static components.
- **Contexts** (`src/lib/contexts/`) — `FileSystemProvider` owns file state + tool-call handling; `ChatProvider` owns messages/input and wires up the AI SDK.

### Generated Code Conventions (enforced by the system prompt)

- Every project must have `/App.jsx` as the root entry point (default export)
- Style with Tailwind CSS, not inline styles
- Non-library imports use the `@/` alias (e.g. `import Button from '@/components/Button'`)
- No HTML files — `App.jsx` is the sole entry point

### Auth

JWT-based sessions (7-day expiry) stored in HTTP-only cookies. `src/lib/auth.ts` handles token creation/verification. `src/middleware.ts` protects routes. Anonymous users can generate components but not persist projects.

### Path Alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).

## Environment Variables

- `ANTHROPIC_API_KEY` — optional; without it the app uses a mock provider that returns static components
- `JWT_SECRET` — defaults to `"development-secret-key"` in dev

## Database

Prisma schema at `prisma/schema.prisma`. Two models: `User` and `Project`. Project stores `messages` and `data` (file system) as JSON strings. Prisma client is generated to `src/generated/prisma/`.
