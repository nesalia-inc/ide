# IDE Project

A local and server-side online IDE with external providers, similar to Sandpack or CodeSandbox but open-source.

## Overview

This project aims to build a browser-based IDE that can run entirely client-side or connect to external runtimes. It draws inspiration from Sandpack and CodeSandbox but focuses on being fully open-source and extensible.

## Architecture

The project is organized as a monorepo with three main packages:

### 1. ide-core (Headless IDE)

The core logic layer - completely UI-agnostic. This package provides:

- **FileSystem**: Virtual file system operations (read, write, mkdir, delete)
- **Terminal**: PTY abstraction with buffer management and resize handling
- **Tabs**: Tab management (open, close, reorder, active state)
- **Workspace**: Project configuration (dependencies, scripts, settings)
- **Process**: Process management (start, stop, input/output streams)

### 2. ide-ui

The user interface layer built with React. Provides:

- Monaco Editor integration with full VS Code-like experience
- FileTree component for browsing project files
- Terminal UI using xterm.js
- Tab bar for open files
- Preview pane with iframe isolation
- Split pane layout system

### 3. ide-compiler

The compilation and execution layer. Supports multiple runtime backends:

- **ClientCompiler**: Uses Web Workers and esbuild for in-browser bundling
- **ServerCompiler**: Connects to external server-side runtimes for non-JavaScript languages

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
├── docs/               # Documentation
├── packages/
│   ├── ide-core/       # Headless IDE logic
│   ├── ide-ui/         # React components
│   └── ide-compiler/   # Compilation backends
```

## License

MIT
