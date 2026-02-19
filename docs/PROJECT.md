# IDE Project

A local and server-side online IDE with external providers, similar to Sandpack or CodeSandbox but open-source.

## Overview

This project aims to build a browser-based IDE that can run entirely client-side or connect to external runtimes. It draws inspiration from Sandpack and CodeSandbox but focuses on being fully open-source and extensible.

## Architecture

The project is organized as a monorepo with two main directories:

### packages/ide-core (Internal Package)

The core package containing all IDE logic - completely UI-agnostic but includes React components. This package provides:

- **FileSystem**: Virtual file system operations (read, write, mkdir, delete)
- **Terminal**: PTY abstraction with buffer management and resize handling
- **Tabs**: Tab management (open, close, reorder, active state)
- **Workspace**: Project configuration (dependencies, scripts, settings)
- **Process**: Process management (start, stop, input/output streams)
- **Compiler**: Client-side (Web Workers + esbuild) and server-side compilation
- **UI Components**: Monaco Editor, FileTree, Terminal UI, Tab bar, Preview

### apps/web (Web Application)

The main web application that consumes `ide-core` to provide a real IDE experience:

- Full IDE interface with layout
- Project management
- Preview in iframe
- Integration with external providers

## Key Features

- **Virtual File System**: In-memory file operations with optional persistence
- **Live Preview**: Real-time preview with hot reload support
- **Multi-language Support**: JavaScript/TypeScript primary, extensible to others
- **Terminal Integration**: Full terminal emulation for running commands
- **Template System**: Pre-configured project templates (React, Vue, Node.js, etc.)
- **Extensible**: Plugin architecture for custom providers and runtimes

## Tech Stack

| Component | Technology |
|-----------|------------|
| UI Framework | React |
| Editor | Monaco Editor |
| Terminal | xterm.js |
| Bundler (client) | esbuild |
| Package Manager | pnpm (monorepo) |

## Getting Started

```bash
# Install dependencies
pnpm install

# Build all packages
pnpm build

# Start development
pnpm dev
```

## Project Structure

```
ide/
├── docs/                   # Documentation
├── packages/
│   └── ide-core/           # Internal package - all IDE logic
├── apps/
│   └── web/                # Webapp consuming ide-core
└── pnpm-workspace.yaml     # Monorepo configuration
```

## License

MIT
