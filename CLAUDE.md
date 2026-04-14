# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe React components in natural language; Claude generates them in real-time using a virtual filesystem with in-browser Babel JSX compilation.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Start dev server with Turbopack (http://localhost:3000)
npm run build       # Production build
npm run lint        # ESLint check
npm run test        # Run all Vitest tests
npm run db:reset    # Reset SQLite database (destructive)
```

**Note:** `dev`, `build`, and `start` scripts prepend `NODE_OPTIONS='--require ./node-compat.cjs'` automatically — this is required for browser-polyfill compatibility.

To run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

### Environment

Copy `.env` and add `ANTHROPIC_API_KEY` to use the real Claude API. Without it, the app runs a `MockLanguageModel` that simulates 4 steps of generation.

## Architecture

### Request Flow

1. User sends a message → `ChatInterface` → `POST /api/chat`
2. `api/chat/route.ts` calls Claude (or mock) with a system prompt + user message + current file state
3. Claude responds with streaming text and calls AI tools (`str_replace_editor`, `file_manager`) to modify virtual files
4. Tool calls are applied to the in-memory `FileSystem` (via `FileSystemContext`)
5. Updated files flow to `PreviewFrame`, which uses Babel (in-browser) to compile and render the JSX
6. If authenticated + projectId present, state is saved to SQLite via server actions

### Key Modules

**`src/lib/file-system.ts`** — Virtual filesystem (in-memory, not disk). Files are stored as a tree and serialized to JSON for DB persistence. All generated React code lives here.

**`src/lib/provider.ts`** — Returns `anthropic('claude-haiku-4-5')` when `ANTHROPIC_API_KEY` is set; otherwise returns a `MockLanguageModel`. Change the model here.

**`src/lib/prompts/generation.tsx`** — System prompt for Claude. Key rules embedded here: always create `/App.jsx` as root, use Tailwind CSS, use `@/` import aliases for internal files, no HTML files.

**`src/lib/tools/`** — Two Vercel AI SDK tools:
- `str_replace_editor`: edits existing file content (str_replace or create commands)
- `file_manager`: creates or deletes files in virtual FS

**`src/app/api/chat/route.ts`** — Streaming chat endpoint using `streamText` from Vercel AI SDK. Receives messages + file state, returns a data stream consumed by `useChat`.

**`src/lib/transform/`** — Babel standalone used client-side to compile JSX before rendering in the preview iframe.

**`src/lib/auth.ts`** — JWT session management using `jose`. Sessions stored in cookies.

**`src/lib/contexts/`** — React contexts:
- `FileSystemContext`: provides virtual FS state to all components
- `ChatContext`: provides chat state and message history

### Data Layer

- **ORM:** Prisma with SQLite (`prisma/dev.db`)
- **Schema:** `User` (id, email, password) and `Project` (id, name, userId, messages JSON, data JSON)
- **Server actions** in `src/actions/` handle DB operations (no API routes for DB — uses Next.js server actions)

### UI Layout

`src/app/main-content.tsx` renders a 3-panel resizable layout:
- Left: Chat panel (`ChatInterface`)
- Right top: Live preview (`PreviewFrame`)
- Right bottom: Code editor (`CodeEditor` with Monaco)

Panel splitting is done with `react-resizable-panels`.

### Path Aliases

`@/*` maps to `./src/*` — use this for all internal imports.

## Testing

Tests use Vitest + React Testing Library with jsdom. Test files live alongside source in `__tests__/` directories. The config is at `vitest.config.mts`.

## Code Style

Use comments sparingly — only for complex or non-obvious logic.
