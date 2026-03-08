# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # Install deps, generate Prisma client, run migrations
npm run dev            # Dev server with Turbopack (localhost:3000)
npm run dev:daemon     # Dev server in background (logs to logs.txt)
npm run build          # Production build
npm run lint           # ESLint (next lint)
npm run test           # Run all tests (vitest)
npx vitest run <path>  # Run a single test file
npm run db:reset       # Reset database (destructive)
```

All dev/build/start scripts require `NODE_OPTIONS='--require ./node-compat.cjs'` (already configured in package.json scripts).

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat interface, Claude generates code via tool calling, and components render live in a sandboxed iframe.

### Data Flow

```
Chat Input → Vercel AI SDK (useChat) → POST /api/chat → Claude (tool calls)
→ Tool responses update VirtualFileSystem → JSX Transform + Import Map
→ Sandboxed iframe preview → Optionally saved to Prisma/SQLite
```

### Key Subsystems

**AI Generation** (`src/app/api/chat/route.ts`, `src/lib/prompts/`, `src/lib/tools/`):
- Uses Claude Haiku 4.5 via `@ai-sdk/anthropic` with tool calling (max 40 steps)
- Two tools: `str_replace_editor` (create/view/replace/insert in files) and `file_manager` (rename/delete)
- Falls back to `MockLanguageModel` in `src/lib/provider.ts` when no `ANTHROPIC_API_KEY` is set
- System prompt enforces `/App.jsx` as entry point and Tailwind CSS for styling

**Virtual File System** (`src/lib/file-system.ts`):
- In-memory file tree with serialization/deserialization — no files written to disk
- Supports create, delete, rename, content replacement, and insertion
- Wrapped by `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) for React state management
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) bridges AI tool calls to file system operations via `handleToolCall()`

**Preview** (`src/components/preview/PreviewFrame.tsx`, `src/lib/transform/jsx-transformer.ts`):
- Transforms JSX/TSX files with Babel standalone, creates blob URLs
- Generates ESM import maps: `@/` aliases resolve locally, third-party packages resolve to `esm.sh` CDN
- CSS files are collected and inlined into the preview HTML

**Auth** (`src/lib/auth.ts`, `src/middleware.ts`):
- JWT sessions (7-day expiry) via `jose`, stored in HTTP-only cookies
- Middleware protects `/api/projects` and `/api/filesystem` routes
- `server-only` import enforces server-side usage

**Persistence** (`prisma/schema.prisma`, `src/actions/`):
- SQLite database with User and Project models
- Messages and file system state stored as JSON strings in the Project model
- Server actions in `src/actions/` handle auth (signUp/signIn/signOut) and project CRUD

### UI Structure

- Path `@/*` maps to `./src/*`
- shadcn/ui components (new-york style) in `src/components/ui/`
- Layout: resizable panels — chat (left) and preview/code editor (right)
- Monaco editor for code editing, markdown rendering for chat messages

### Tech Stack

- Next.js 15 (App Router, Turbopack), React 19, TypeScript
- Tailwind CSS v4, shadcn/ui (Radix UI + lucide-react)
- Prisma + SQLite, Vercel AI SDK
- Vitest + Testing Library + jsdom for tests
