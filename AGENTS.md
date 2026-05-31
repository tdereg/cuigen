# UIGen — Agent Instructions

AI-powered React component generator with live preview. Claude generates components via tool calls into an in-memory virtual file system; no real files are written to disk.

## Commands

```bash
npm run setup        # First-time: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (Turbopack) at http://localhost:3000
npm run build        # Production build
npm run test         # Run Vitest unit tests
npm run lint         # ESLint
npm run db:reset     # Reset SQLite database (destructive)
```

> **Never run `npm audit fix`** — dependencies are pinned to specific compatible versions.

## Architecture

### AI Tool Loop
Claude interacts with the app exclusively through two tools defined in `src/lib/tools/`:
- `str_replace_editor` — create or patch file content in the virtual FS
- `file_manager` — manage directory structure

The system prompt is in `src/lib/prompts/generation.tsx`. Prompt caching is enabled (ephemeral).

### Virtual File System
All generated component code lives in an in-memory `VirtualFileSystem` class (`src/lib/file-system.ts`). It serializes to JSON and is persisted in the `Project.data` Prisma field for authenticated users. Never write component code to disk.

### State Management
Two React contexts wrap the main UI:
- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — virtual FS state
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — chat messages and streaming

### Auth
JWT tokens in `httpOnly` cookies (7-day expiry). Auth utilities live in `src/lib/auth.ts` marked `server-only`. Anonymous users can use the app without signup; projects have an optional `userId`.

### Mock Provider
If `ANTHROPIC_API_KEY` is unset or set to the placeholder `your-api-key-here`, the app falls back to `MockLanguageModel` in `src/lib/provider.ts` which returns canned components. This is intentional — don't break this fallback.

## Gotchas

**Dual tool execution** — tools run server-side in `route.ts` AND are replayed client-side in `file-system-context.tsx`. Both paths modify the virtual FS independently. Don't assume a single code path.

**Tool registration asymmetry** — `str_replace_editor` (`src/lib/tools/str-replace.ts`) is a raw object `{ id, args, parameters, execute }`. `file_manager` uses the `ai` SDK `tool()` wrapper. They have different parameter validation behavior.

**Mock provider args are stringified** — in `MockLanguageModel` (`src/lib/provider.ts`), tool `args` are `JSON.stringify(...)`. The real provider passes objects. Context at line 145 of `file-system-context.tsx` expects objects — mock output must be parsed first.

**CSS imports stripped** — `jsx-transformer.ts` removes all CSS `import` statements before Babel transform and injects styles separately into an iframe `<style>` tag. CSS-in-JS patterns that rely on import side-effects won't work.

**Mixed indexing in VirtualFileSystem** — `viewFile(path, [start, end])` uses 1-based line numbers; `insertInFile(path, content, insertLine)` uses 0-based. Don't reuse the same index across both calls.

**Import map aliasing** — each virtual file is registered under 8+ path variants (with/without leading `/`, with/without extension, with/without `@/` prefix). Adding a 9th variant bloats the map. Missing a variant causes silent import failures in the preview iframe.

**Entry point search order** — `PreviewFrame.tsx` looks for `/App.jsx`, `/App.tsx`, `/index.jsx`, `/index.tsx`, `/src/App.jsx`, `/src/App.tsx` in that order. Adding `index.jsx` can silently shadow `App.jsx`.

**System message mutated via unshift** — `route.ts` calls `messages.unshift(systemMsg)` on the input array. Reusing the same array in a second call will prepend the system message twice.

## Key Directories

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | AI streaming endpoint |
| `src/lib/tools/` | Tool definitions for Claude |
| `src/lib/prompts/` | System prompt |
| `src/lib/file-system.ts` | VirtualFileSystem class |
| `src/components/preview/` | Live preview renderer |
| `src/components/editor/` | Monaco-based code editor |
| `src/actions/` | Next.js Server Actions |
| `prisma/schema.prisma` | DB schema (SQLite) |

## Conventions

- Path aliases: `@/*` → `src/*`
- UI components: shadcn/ui (New York style) via `npx shadcn add <component>`
- Icons: Lucide React
- Utility: `cn()` from `@/lib/utils` (clsx + tailwind-merge)
- Interactive components require `"use client"`; auth/DB utilities use `"server-only"`
- TypeScript strict mode is on

## Environment

- `ANTHROPIC_API_KEY` — set in `.env`; leave as placeholder to use mock mode
- `JWT_SECRET` — defaults to `"development-secret-key"` if unset
- Database: SQLite at `prisma/dev.db`
