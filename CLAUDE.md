# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, and the AI generates React code that's displayed in a live preview. The application uses a virtual file system (no files written to disk) and supports both authenticated and anonymous usage.

## Development Commands

### Setup
```bash
npm run setup              # Install dependencies, generate Prisma client, run migrations
```

### Development
```bash
npm run dev                # Start Next.js dev server with Turbopack
npm run dev:daemon         # Start server in background, logs to logs.txt
```

### Testing
```bash
npm test                   # Run all tests with Vitest
```

### Linting & Building
```bash
npm run lint               # Run ESLint
npm run build              # Build production bundle
npm start                  # Start production server
```

### Database
```bash
npx prisma generate        # Generate Prisma client (output: src/generated/prisma)
npx prisma migrate dev     # Create and apply migration
npm run db:reset           # Reset database (warning: deletes all data)
npx prisma studio          # Open database GUI
```

## Architecture

### AI Chat Flow
1. User sends message via `ChatInterface` component
2. Request hits `/api/chat/route.ts` with messages and current file state
3. System prompt from `src/lib/prompts/generation.tsx` is prepended
4. Request streams to Claude API (or mock provider if no API key)
5. AI uses tools (`str_replace_editor`, `file_manager`) to manipulate virtual file system
6. File changes trigger live preview updates
7. On completion, project state is saved to database (if authenticated)

### Virtual File System
- **Core**: `src/lib/file-system.ts` - In-memory file system using Map data structures
- Files never touch disk during generation/editing
- Serializes to JSON for storage in Prisma database (`Project.data` field)
- Deserializes on load to reconstruct file tree

### AI Provider System
- **Provider**: `src/lib/provider.ts` - Exports `getLanguageModel()`
- If `ANTHROPIC_API_KEY` is set: uses `claude-haiku-4-5` via `@ai-sdk/anthropic`
- If no API key: uses `MockLanguageModel` that returns static code snippets
- Mock provider prevents repetitive tool calls by limiting `maxSteps: 4`

### AI Tools
Tools are defined in `src/lib/tools/`:
- `str-replace.ts`: Edit existing files via find-and-replace
- `file-manager.ts`: Create, delete, list, view files in virtual file system

### Authentication & Sessions
- JWT-based sessions using `jose` library
- Session tokens stored in HTTP-only cookies
- Password hashing with bcrypt
- Middleware protects `/api/projects` and `/api/filesystem` routes
- Anonymous users can use app but can't persist projects

### Database Schema
- **User**: Email/password authentication, has many projects
- **Project**: Contains `messages` (JSON array) and `data` (serialized file system)
- Prisma client generated to `src/generated/prisma/`
- SQLite database at `prisma/dev.db`

### Component Structure
- **Chat**: `src/components/chat/` - Message handling, input, rendering
- **Editor**: `src/components/editor/` - Code view with syntax highlighting
- **Preview**: `src/components/preview/` - Live component preview with Babel transpilation
- **Auth**: `src/components/auth/` - Sign in/up dialogs
- UI components: Radix UI primitives + Tailwind CSS v4

### Context Providers
- `ChatContext` (`src/lib/contexts/chat-context.tsx`): Manages message history and streaming
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`): Manages virtual file system state across components

### Server Actions
Located in `src/actions/`:
- `signUp`, `signIn`, `signOut`, `getUser` - Authentication
- `createProject`, `getProject`, `getProjects` - Project management

## Key Technical Details

- **Next.js 15**: App Router with React Server Components
- **React 19**: Concurrent rendering features
- **Tailwind CSS v4**: Using new PostCSS plugin (`@tailwindcss/postcss`)
- **Vercel AI SDK**: Handles streaming responses and tool calls
- **Monaco Editor**: Code editing interface
- **Babel Standalone**: Client-side JSX transpilation for preview
- **Vitest**: Testing with React Testing Library and jsdom

## Testing Strategy
Test files colocated with components in `__tests__` directories. Example: `src/components/chat/__tests__/MessageList.test.tsx`

## Anonymous User Tracking
`src/lib/anon-work-tracker.ts` assigns temporary IDs to anonymous users for tracking work sessions without authentication.
