# Contributing to UIGen

Welcome! UIGen is an AI-powered React component generator with live preview. Whether you're fixing a typo, adding a feature, or improving docs, your contributions are appreciated.

This guide walks you through everything you need to get started — even if this is your first open-source contribution.

## Prerequisites

Make sure you have the following installed on your machine:

- **Node.js 18+** — [download here](https://nodejs.org/). Run `node -v` to check your version.
- **npm** — comes bundled with Node.js. Run `npm -v` to verify.
- **Git** — [download here](https://git-scm.com/). Run `git --version` to verify.

## Getting Started

### 1. Fork and clone the repository

Click the **Fork** button on the GitHub repo page to create your own copy, then clone it locally:

```bash
git clone https://github.com/<your-username>/uigen.git
cd uigen
```

### 2. Run the setup script

```bash
npm run setup
```

This single command does three things:

1. **Installs dependencies** (`npm install`) — downloads all required packages
2. **Generates the Prisma client** (`npx prisma generate`) — creates the database client code from the schema
3. **Runs database migrations** (`npx prisma migrate dev`) — creates the local SQLite database at `prisma/dev.db`

### 3. (Optional) Add your Anthropic API key

Edit the `.env` file in the project root:

```
ANTHROPIC_API_KEY=your-api-key-here
```

This is **optional**. Without an API key the app uses a mock provider that returns static components, so you can develop and test most features without one.

### 4. Start the development server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser. You should see the UIGen interface.

## Project Structure

Here's an overview of the directory layout:

```
uigen/
├── .github/workflows/     # CI pipeline (lint, test, build)
│   └── ci.yml
├── prisma/                 # Database layer
│   ├── schema.prisma       # Prisma schema (User and Project models)
│   ├── migrations/         # Database migration files
│   └── dev.db              # Local SQLite database (gitignored)
├── src/
│   ├── actions/            # Next.js Server Actions
│   │   ├── create-project.ts
│   │   ├── get-project.ts
│   │   └── get-projects.ts
│   ├── app/                # Next.js App Router
│   │   ├── layout.tsx      # Root layout
│   │   ├── page.tsx        # Home page
│   │   ├── [projectId]/    # Dynamic project routes
│   │   └── api/chat/       # Chat API route (POST /api/chat)
│   │       └── route.ts
│   ├── components/         # React components
│   │   ├── auth/           # Authentication UI (sign in, sign up dialogs)
│   │   ├── chat/           # Chat interface (messages, input, markdown)
│   │   ├── editor/         # Code editor (Monaco) and file tree
│   │   ├── preview/        # Live preview iframe
│   │   └── ui/             # Reusable UI primitives (shadcn/ui + Radix)
│   ├── hooks/              # Custom React hooks
│   │   └── use-auth.ts
│   ├── lib/                # Core logic
│   │   ├── auth.ts         # JWT session management
│   │   ├── contexts/       # React Contexts (ChatProvider, FileSystemProvider)
│   │   ├── file-system.ts  # Virtual File System implementation
│   │   ├── prisma.ts       # Prisma client singleton
│   │   ├── prompts/        # System prompt for the AI
│   │   ├── provider.ts     # AI provider (Anthropic or mock)
│   │   ├── tools/          # AI tool definitions (str_replace_editor, file_manager)
│   │   ├── transform/      # JSX-to-JS transformer (Babel Standalone)
│   │   └── utils.ts        # Shared utility functions
│   ├── generated/prisma/   # Generated Prisma client (gitignored, do not edit)
│   └── middleware.ts       # Route protection middleware
├── package.json
├── tsconfig.json
└── vitest.config.mts
```

## Architecture Overview

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15 (App Router) |
| UI | React 19, Tailwind CSS v4 |
| Language | TypeScript |
| Database | Prisma + SQLite |
| AI | Anthropic Claude via Vercel AI SDK |
| Code Editor | Monaco Editor |
| Testing | Vitest + React Testing Library |

### Request Flow

Here's what happens when a user sends a chat message:

```
User types prompt
       │
       ▼
ChatInterface (React component)
       │
       ▼
ChatProvider calls POST /api/chat (via Vercel AI SDK's useChat)
       │
       ▼
API route reconstructs VirtualFileSystem from request body,
attaches system prompt + AI tools, calls streamText with Claude
       │
       ▼
Streamed tool calls arrive on the client
       │
       ▼
FileSystemContext applies tool calls to the in-memory virtual file system
       │
       ▼
PreviewFrame transforms JSX → JS with Babel and renders in an iframe
```

### Key Concepts

- **Virtual File System** (`src/lib/file-system.ts`) — An in-memory file tree with CRUD operations. All file paths are `/`-prefixed (e.g., `/App.jsx`, `/components/Button.jsx`). The entire file system is serialized as JSON and sent with every chat request.

- **AI Tools** (`src/lib/tools/`) — Two tools are exposed to Claude:
  - `str_replace_editor` — create, view, and edit files via string replacement
  - `file_manager` — list, rename, and delete files

- **Provider** (`src/lib/provider.ts`) — Returns the Anthropic Claude model when `ANTHROPIC_API_KEY` is set. Otherwise falls back to a `MockLanguageModel` that returns static components.

- **Path Alias** — `@/*` maps to `./src/*` (configured in `tsconfig.json`). Use this for all imports within the project (e.g., `import { cn } from "@/lib/utils"`).

- **Root Component** — Every generated project must have `/App.jsx` as the root entry point with a default export.

## Development Commands

| Command | Description |
|---------|-------------|
| `npm run setup` | Install dependencies, generate Prisma client, run migrations |
| `npm run dev` | Start the dev server (Next.js + Turbopack) |
| `npm run build` | Create a production build |
| `npm start` | Start the production server |
| `npm run lint` | Run ESLint |
| `npm test` | Run tests in watch mode (Vitest) |
| `npm test -- --run` | Run tests once (no watch mode — used in CI) |
| `npm run db:reset` | Reset the SQLite database |
| `npx prisma generate` | Regenerate Prisma client after schema changes |
| `npx prisma migrate dev` | Run pending database migrations |

## Code Style & Conventions

- **TypeScript** is used throughout the project. Avoid `any` types where possible.
- **ESLint** — Run `npm run lint` to check for issues. CI enforces this.
- **Tailwind CSS** — Use Tailwind utility classes for all styling. Avoid inline styles.
- **Import alias** — Use `@/` for imports (e.g., `import { cn } from "@/lib/utils"`), not relative paths like `../../lib/utils`.
- **UI components** — Reusable primitives live in `src/components/ui/` (built on Radix UI via shadcn/ui).

## Testing

The project uses **Vitest** with **React Testing Library**.

### Running tests

```bash
# Watch mode — re-runs tests as you save files
npm test

# Single run — useful before committing
npm test -- --run
```

### Where tests live

Tests are colocated with their source files in `__tests__/` folders:

```
src/components/chat/__tests__/ChatInterface.test.tsx
src/lib/__tests__/file-system.test.ts
src/lib/contexts/__tests__/chat-context.test.tsx
```

When adding new functionality, add tests in a `__tests__/` directory next to the source file.

## Submitting Changes

### 1. Create a feature branch

Always branch off `master`:

```bash
git checkout master
git pull origin master
git checkout -b my-feature-name
```

### 2. Make your changes

Write code, add tests if applicable, and make sure everything works:

```bash
npm run lint       # Check for lint errors
npm test -- --run  # Run tests
npm run build      # Verify the build succeeds
```

### 3. Commit with a clear message

```bash
git add <files-you-changed>
git commit -m "Add description of what you changed"
```

Write commit messages that explain **what** changed and **why**.

### 4. Push and open a pull request

```bash
git push origin my-feature-name
```

Then open a pull request on GitHub against the `master` branch. In your PR description, explain what you changed and why.

### 5. Wait for CI and review

CI will automatically run **lint**, **tests**, and **build** — all three must pass. A maintainer will review your PR and may request changes.

## Troubleshooting

### Prisma errors

If you see errors related to Prisma or the database:

```bash
# Regenerate the Prisma client
npx prisma generate

# Reset the database entirely (deletes all data)
npm run db:reset
```

### Node.js version issues

Make sure you're running Node.js 18 or higher:

```bash
node -v
# Should output v18.x.x or higher
```

If you're using a version manager like [nvm](https://github.com/nvm-sh/nvm), switch to the correct version:

```bash
nvm use 18
```

### Port already in use

If port 3000 is occupied, Next.js will automatically try the next available port. Check the terminal output for the actual URL.

---

Thank you for contributing!
