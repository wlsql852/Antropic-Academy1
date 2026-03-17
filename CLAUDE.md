# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack (localhost:3000)
npm run build        # Production build
npm run test         # Run Vitest unit tests
npm run lint         # Run ESLint
npm run db:reset     # Reset SQLite database (destructive)
npx prisma migrate dev    # Run new migrations
npx prisma studio         # Open Prisma Studio GUI
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already included in the npm script).

Set `ANTHROPIC_API_KEY` in `.env` to use the real Claude API. If empty/unset, the app falls back to `MockLanguageModel` in `src/lib/provider.ts` which returns predefined component examples.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates them via tool calls that mutate an in-memory virtual file system, which is then transpiled and rendered in an iframe.

### Request Flow

1. User sends message → `ChatContext` (`src/lib/contexts/chat-context.tsx`) via Vercel AI SDK `useChat`
2. POST to `/api/chat` (`src/app/api/chat/route.ts`) — streams Claude response with tool calls
3. Claude calls `str_replace_editor` / `file_manager` tools to create/edit virtual files
4. Tool results update `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`)
5. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) detects FS changes, calls `JSXTransformer`
6. `JSXTransformer` (`src/lib/transform/jsx-transformer.ts`) transpiles JSX with Babel, builds an import map using esm.sh CDN, injects Tailwind CDN, and generates an HTML blob URL rendered in an iframe

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is the core abstraction — a fully in-memory FS with no disk writes. `/App.jsx` is the entry point for preview. The FS is serialized as JSON into `Project.data` in SQLite for persistence.

### AI Tools

Defined in `src/lib/tools/`:
- **`str_replace_editor`** — `view`, `create`, `str_replace`, `insert`, `undo_edit` commands on virtual files
- **`file_manager`** — `rename`, `delete` operations

Tools use Zod schema validation and are passed to the Vercel AI SDK `streamText` call.

### Preview Rendering

`JSXTransformer` handles Babel transpilation of JSX → ES modules, builds an import map (esm.sh URLs for npm packages), injects Tailwind CSS via CDN script, wraps the component in an error boundary, and creates a `blob:` URL for the iframe `src`. Missing imports get placeholder stub modules.

### Authentication

JWT tokens (`jose` library) stored in httpOnly cookies, 7-day expiry. Passwords hashed with bcrypt. `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes. Auth server actions are in `src/actions/index.ts`. Anonymous users can use the app without signing in; their work is tracked via localStorage (`src/lib/anon-work-tracker.ts`).

### Database

Prisma + SQLite (`prisma/dev.db`). Schema is defined in `prisma/schema.prisma` — reference it to understand stored data structure. Two models: `User` (email/bcrypt password) and `Project` (stores serialized virtual FS in `data` JSON field + chat history in `messages` JSON field, optional `userId` to support anonymous projects).

## Code Style

Use comments sparingly — only on complex or non-obvious logic.

### Key Paths

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | Streaming chat endpoint, Claude integration, DB persistence |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/lib/transform/jsx-transformer.ts` | Babel transpile + preview HTML generation |
| `src/lib/provider.ts` | Anthropic/Mock model provider switch |
| `src/lib/prompts/generation.tsx` | System prompt guiding AI to generate Tailwind React components |
| `src/lib/contexts/file-system-context.tsx` | FS state + tool call handlers |
| `src/lib/contexts/chat-context.tsx` | Chat state, useChat integration |
| `src/app/main-content.tsx` | Top-level UI layout (resizable panels: chat 35% / preview+editor 65%) |
