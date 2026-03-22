# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies and set up database
npm run setup

# Development (uses Turbopack)
npm run dev

# Build for production
npm run build

# Run tests (Vitest)
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Database
npx prisma migrate dev     # Apply migrations
npm run db:reset           # Reset database
npx prisma studio          # GUI for the database
```

## Environment

- `ANTHROPIC_API_KEY` — optional; without it the app uses `MockLanguageModel` (defined in `src/lib/provider.ts`)
- `JWT_SECRET` — defaults to `"development-secret-key"`
- SQLite database lives at `prisma/dev.db`

The `node-compat.cjs` shim is required via `NODE_OPTIONS` in all scripts to remove Web Storage API globals incompatible with Node.js 25+ SSR.

## Architecture

UIGen is a Next.js 15 (App Router) app where users describe React components in natural language and get live previews powered by Claude AI.

### Data flow

1. User sends a message → `/api/chat/route.ts` streams a response from Claude via Vercel AI SDK
2. Claude calls tools (`str_replace_editor`, `file_manager`) to create/edit files in the virtual file system
3. Tool results update the virtual FS state held in `FileSystemContext`
4. `PreviewFrame` recompiles changed files with Babel standalone and re-renders the iframe

### Virtual file system (`src/lib/file-system.ts`)

All "files" live in memory — nothing is written to disk. The `VirtualFileSystem` class handles create, update, delete, rename, and import alias resolution (`@/` → `src/`). It serializes to JSON for database persistence (stored in the `Project.data` column).

### Tool system (`src/lib/tools/`)

Claude uses two tools during generation:
- `str_replace_editor` — creates files or performs targeted string replacements/insertions (like a minimal editor)
- `file_manager` — renames or deletes files/directories

Tool definitions and handlers are wired into the chat route and update the file system context via callbacks.

### Preview (`src/components/preview/PreviewFrame.tsx`, `src/lib/transform/jsx-transformer.ts`)

Runs entirely client-side. `jsx-transformer.ts` uses `@babel/standalone` to compile JSX and builds an import map that resolves virtual files and CDN-hosted packages. The compiled output is injected into a sandboxed iframe.

### Persistence

Anonymous users: work is tracked in localStorage via `src/lib/anon-work-tracker.ts`.
Authenticated users: projects (chat messages + serialized FS) are saved to SQLite through Server Actions in `src/actions/`.

### Contexts

- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — owns all virtual FS state and exposes file operations to the component tree
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — owns chat history and wires Vercel AI SDK `useChat` to the file system context

### Auth

JWT sessions (7-day, `jose` library). `src/middleware.ts` protects `/[projectId]` routes. `src/lib/auth.ts` handles token sign/verify. Passwords are hashed with bcrypt.

### System prompt

`src/lib/prompts/generation.tsx` contains the full system prompt sent to Claude, including instructions on how to use the two tools. Changing this file directly affects generation behavior.
