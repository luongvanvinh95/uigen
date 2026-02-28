# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup       # First-time setup: install deps, generate Prisma client, run migrations
npm run dev         # Dev server with Turbopack on localhost:3000
npm run build       # Production build
npm run lint        # ESLint
npm run test        # Vitest in watch mode
npm run test -- --run  # Run tests once (CI mode)
npm run db:reset    # Reset SQLite database
```

Run a single test file:
```bash
npm run test -- src/lib/__tests__/file-system.test.ts
```

## Environment

Copy `.env` and optionally set `ANTHROPIC_API_KEY`. Without a key, the app falls back to a mock provider (no real AI responses).

## Architecture

UIGen is a Next.js 15 App Router app where users describe React components in a chat, Claude generates code, and a live preview renders the result instantly.

### Virtual File System (VFS)

The central concept is an **in-memory virtual file system** (`src/lib/file-system.ts`). No generated files are ever written to disk. The VFS:
- Stores files as a `Map<string, string>` keyed by path
- Serializes to JSON for persistence in the `Project.data` DB column
- Supports CRUD + rename, plus text editor operations (`viewFile`, `replaceInFile`, `insertInFile`)

### AI Tool Integration

The chat API route (`src/app/api/chat/route.ts`) exposes two tools to Claude:
- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): create/read/edit files via string replacement
- **`file_manager`** (`src/lib/tools/file-manager.ts`): rename and delete files

When Claude calls these tools, the results are intercepted client-side in `ChatContext` via `handleToolCall`, which applies the operations to the VFS held in `FileSystemContext`. The VFS state is serialized and sent back with each subsequent chat message so the API route can reconstruct it.

### State Management

Two React contexts wire everything together:
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): owns VFS state; exposes file operations and a refresh trigger for the preview
- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): wraps Vercel AI SDK's `useChat`; forwards tool call results to FileSystemContext

Both are provided in `src/app/main-content.tsx`, which also defines the resizable panel layout (35% chat / 65% preview+code).

### Live Preview

`src/components/preview/PreviewFrame.tsx` renders components in a sandboxed `<iframe>`. It:
1. Picks the entry file (looks for `App.jsx/tsx`, `index.jsx/tsx`, etc.)
2. Passes all VFS files through `src/lib/transform/jsx-transformer.ts` (Babel standalone for JSX → JS)
3. Uses an import map so generated components can `import` each other by path

### Persistence

- **Authenticated users**: Projects saved to SQLite via Prisma after each AI response completes (`onFinish` in the API route)
- **Anonymous users**: Work tracked in localStorage via `src/lib/anon-work-tracker.ts`
- **Schema**: `Project` has `messages` (JSON array) and `data` (serialized VFS JSON) columns; `userId` is optional to support anonymous projects

### Authentication

JWT sessions via `jose` stored in an httpOnly cookie. `src/lib/auth.ts` handles `createSession`/`getSession`/`verifySession`. `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes. Server actions call `getSession()` before any DB operation.

### AI Provider

`src/lib/provider.ts` exports `getLanguageModel()`. If `ANTHROPIC_API_KEY` is set, it returns the real Anthropic model (`claude-haiku-4-5`). Otherwise it returns a mock provider. The system prompt with prompt caching lives in `src/lib/prompts/generation.tsx`.

## Key Conventions

- Path alias `@/*` maps to `src/*`
- shadcn/ui components live in `src/components/ui/` (New York style, neutral base color, CSS variables)
- Server actions in `src/actions/` (re-exported from `src/actions/index.ts`)
- Tests co-located in `__tests__/` subdirectories next to the code they test
