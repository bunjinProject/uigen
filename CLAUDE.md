# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (with Turbopack)
npm run dev

# Run tests
npm test

# Run a single test file
npx vitest run src/path/to/test.test.ts

# Run tests in watch mode
npx vitest

# Lint
npm run lint

# Build for production
npm run build

# Reset database
npm run db:reset
```

## Architecture

UIGen is an AI-powered React component generator built with Next.js 15 (App Router) and React 19. Users describe components in a chat interface, and Claude generates code that renders in a live preview.

### Core Data Flow

1. User sends message via chat interface
2. `POST /api/chat` receives message with current virtual file system state
3. Claude uses `str_replace_editor` and `file_manager` tools to create/edit files
4. Tool calls stream back to client and update `FileSystemContext`
5. `PreviewFrame` transforms JSX files and renders them in a sandboxed iframe

### Key Abstractions

**Virtual File System (`src/lib/file-system.ts`)**: In-memory file system that stores all generated code. No files are written to disk. Supports standard operations (create, read, update, delete, rename) plus editor-specific commands (view with line numbers, str_replace, insert at line).

**JSX Transformer (`src/lib/transform/jsx-transformer.ts`)**: Transforms JSX/TSX files using Babel standalone. Creates blob URLs for each file and an import map so the browser can resolve imports. Third-party packages are loaded from esm.sh. CSS files are extracted and injected as style tags.

**Context Providers**:
- `FileSystemContext`: Manages virtual file system state and handles tool calls from AI
- `ChatContext`: Wraps Vercel AI SDK's `useChat` hook, sends file system state with each request

### AI Tool System

Two tools are available to Claude for code generation:

- `str_replace_editor`: view, create, str_replace, insert commands for file editing
- `file_manager`: rename, delete commands for file management

Tools are defined in `src/lib/tools/` and executed server-side. Tool calls are also replayed client-side in `FileSystemContext.handleToolCall()` to keep UI in sync.

### Preview System

Preview works by:
1. Transforming all JSX/TSX files to ES modules via Babel
2. Creating blob URLs for each transformed file
3. Building an import map that resolves local imports to blob URLs and packages to esm.sh
4. Rendering in an iframe with sandbox restrictions (allow-scripts, allow-same-origin, allow-forms)
5. Tailwind CSS is loaded from CDN in the preview

### Authentication

JWT-based auth using jose library. Sessions stored in cookies. Anonymous users can use the app; authenticated users get persistent projects stored in SQLite via Prisma.

### Database

SQLite database managed by Prisma. Models:
- `User`: email, password (hashed with bcrypt)
- `Project`: name, messages (JSON), data (serialized file system), optional user reference

Prisma client is generated to `src/generated/prisma`.

## Generated Code Conventions

When Claude generates React components for users:
- Entry point must be `/App.jsx` with default export
- Use Tailwind CSS for styling, not inline styles
- Import local files with `@/` alias (e.g., `@/components/Button`)
- No HTML files needed - App.jsx is the entry point
