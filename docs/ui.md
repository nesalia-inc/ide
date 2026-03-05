# UI Layer Architecture

## Overview

The UI Layer connects the headless VFS to React components. It provides reactive components that automatically sync with the virtual file system.

## Architecture

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│     VFS         │      │   UI Bridge    │      │  React UI      │
│   (headless)    │ ───► │   (hooks)      │ ───► │  Components    │
│                 │      │                │      │                │
│ - writeFile()   │      │ - subscribe()  │      │ - FileTree     │
│ - events        │      │ - useFile()    │      │ - FileExplorer │
└─────────────────┘      │ - useWatcher() │      │ - CodeEditor   │
                          └─────────────────┘      └─────────────────┘
```

## Sync Strategy: Push via Events

The UI uses a **push-based** approach with VFS events. Components subscribe to file system changes and update automatically.

```ts
// Components receive real-time updates
const FileExplorer = () => {
  const files = useVFS()

  return (
    <Tree>
      {files.map(f => <FileNode key={f.path} file={f} />)}
    </Tree>
  )
}
```

## Required Hooks

| Hook | Description | Returns |
|------|-------------|---------|
| `useVFS` | Access to VFS instance | `VFS` |
| `useFile(path)` | Subscribe to file changes | `{ content, isLoading, error, update }` |
| `useDirectory(path)` | Subscribe to directory listing | `{ files, isLoading, error }` |
| `useWatcher(path, options)` | Subscribe to file events | `{ events, unsubscribe }` |
| `useLock(path)` | Manage file locks | `{ lock, isLocked, lock, unlock }` |
| `useTabs()` | Manage open tabs | `{ tabs, open, close, setActive }` |
| `useSearch(query)` | Search files | `{ results, isSearching }` |

## Components

| Component | Responsibility | Dependencies |
|-----------|---------------|--------------|
| `FileTree` | Expandable tree view | `readDir`, `watch`, `stat` |
| `FileExplorer` | Sidebar with actions (new, rename, delete) | All VFS methods |
| `CodeEditor` | Monaco editor wrapper | `readFile`, `writeFile`, `lock` |
| `Tabs` | Tab management (editor/terminal/preview) | `stat`, `exists` |
| `Breadcrumb` | Path navigation | `resolve`, `dirname` |
| `ContextMenu` | Right-click menu | `stat`, `getLock`, `lock` |
| `SearchBar` | File search | `readDir` + glob |
| `StatusBar` | Show current file, locks, errors | Various |

## State Management

We recommend **Zustand** for state management:

```ts
import { create } from 'zustand'

type IDEState = {
  vfs: VFS
  tabs: Tab[]
  activeTabId: string | null
  setVFS: (vfs: VFS) => void
  openTab: (tab: Tab) => void
  closeTab: (id: string) => void
  setActiveTab: (id: string) => void
}

export const useIDEStore = create<IDEState>((set) => ({
  tabs: [],
  activeTabId: null,
  setVFS: (vfs) => set({ vfs }),
  openTab: (tab) => set((state) => ({
    tabs: [...state.tabs, tab]
  })),
  closeTab: (id) => set((state) => ({
    tabs: state.tabs.filter(t => t.id !== id)
  })),
  setActiveTab: (id) => set({ activeTabId: id })
}))
```

## Optimistic Updates

UI should update immediately on user action, then sync with VFS asynchronously:

```ts
const Editor = ({ path }) => {
  const file = useFile(path)

  const handleChange = (content) => {
    // 1. Optimistic update - UI updates immediately
    setLocalContent(content)

    // 2. Async VFS write
    vfs.writeFile(path, content)
      .then(() => setSaved(true))
      .catch((error) => {
        // 3. Rollback on error
        setLocalContent(file.content)
        setError(error.message)
      })
  }

  return (
    <CodeEditor
      value={localContent}
      onChange={handleChange}
    />
  )
}
```

## Performance Considerations

### Virtualization

For large directories (1000+ files), use virtualization:

```ts
import { useVirtualizer } from '@tanstack/react-virtual'

const FileList = ({ files }) => {
  const parentRef = useRef(null)

  const virtualizer = useVirtualizer({
    count: files.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 32,
  })

  return (
    <div ref={parentRef} style={{ height: '100%', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize() }}>
        {virtualizer.getVirtualItems().map((virtualRow) => (
          <FileItem
            key={virtualRow.key}
            file={files[virtualRow.index]}
            style={{ position: 'absolute', top: virtualRow.start }}
          />
        ))}
      </div>
    </div>
  )
}
```

### Lazy Loading

Only load file contents when needed:

```ts
// Don't load all files in memory
const useDirectory = (path: string) => {
  const [files, setFiles] = useState<DirectoryEntry[]>([])
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    // Only fetch when component mounts or path changes
    vfs.readDir(path).then((entries) => {
      setFiles(entries)
      setIsLoading(false)
    })
  }, [path])

  return { files, isLoading }
}
```

## Conflict Resolution

When multiple users edit the same file:

```ts
const useFileLock = (path: string) => {
  const { lock, isLocked } = useLock(path)

  const handleEdit = async (content: string) => {
    if (isLocked && lock?.owner !== currentUser) {
      // Show conflict dialog
      showConflictDialog({
        file: path,
        lockedBy: lock.owner,
        theirContent: lock.content,
        yourContent: content,
        onMerge: (merged) => vfs.writeFile(path, merged),
        onOverwrite: () => vfs.writeFile(path, content),
        onCancel: () => {}
      })
      return
    }

    // Lock and write
    await vfs.lock(path, currentUser, { write: true })
    await vfs.writeFile(path, content)
  }
}
```

## Drag and Drop

For moving files between directories:

```ts
const handleDrop = async (draggedPath: string, targetPath: string) => {
  // Check locks before move
  const lock = vfs.getLock(draggedPath)

  if (lock && lock.permissions?.rename) {
    showError('File is locked')
    return
  }

  // Perform move
  const result = vfs.move(draggedPath, targetPath)

  if (isErr(result)) {
    showError(result.error)
  }
}
```

## Styling

Components use **CSS variables** for theming and are **Tailwind-compatible**:

### CSS Variables

```css
/* Base theme */
:root {
  --ide-bg: #1e1e1e;
  --ide-fg: #d4d4d4;
  --ide-accent: #007acc;
  --ide-border: #3c3c3c;
  --ide-sidebar-width: 250px;
  --ide-tab-height: 35px;
}

/* Dark theme */
[data-theme="dark"] {
  --ide-bg: #1e1e1e;
  --ide-fg: #d4d4d4;
}

/* Light theme */
[data-theme="light"] {
  --ide-bg: #ffffff;
  --ide-fg: #1e1e1e;
}
```

### Tailwind Compatibility

```ts
// Components use utility classes that work with Tailwind
import { cn } from '@deessejs/ide-ui/lib/utils'

const Component = ({ className }: { className?: string }) => {
  return (
    <div className={cn(
      "flex items-center gap-2 p-2",
      "bg-[var(--ide-bg)] text-[var(--ide-fg)]",
      className
    )}>
      {children}
    </div>
  )
}

// Override any style via className prop
<FileTree className="w-64 bg-red-500" />
```

### No Tailwind? No Problem

CSS variables work without Tailwind:

```css
.ide-container {
  background-color: var(--ide-bg);
  color: var(--ide-fg);
  padding: var(--ide-spacing, 8px);
}
```

```css
:root {
  --ide-bg: #1e1e1e;
  --ide-fg: #d4d4d4;
  --ide-accent: #007acc;
  --ide-border: #3c3c3c;
  --ide-sidebar-width: 250px;
  --ide-tab-height: 35px;
}
```

Components use these variables:

```ts
const styles = {
  container: {
    backgroundColor: 'var(--ide-bg)',
    color: 'var(--ide-fg)',
    borderColor: 'var(--ide-border)'
  }
}
```

## Component Philosophy (shadcn-like)

Each component is **independent**, **composable**, and can be imported individually. Like shadcn, you only import what you need.

### Import Pattern

```ts
// Import individual components - tree-shakeable
import { FileTree } from '@deessejs/ide-ui/FileTree'
import { CodeEditor } from '@deessejs/ide-ui/CodeEditor'
import { Tabs } from '@deessejs/ide-ui/Tabs'
import { Breadcrumb } from '@deessejs/ide-ui/Breadcrumb'

// Each component works standalone
// No need to import the whole UI library
```

### Each Component Includes Its Own Logic

```ts
// FileTree.tsx - self-contained
import { useDirectory } from '@deessejs/ide-ui/hooks/useDirectory'
import { FileTreeItem } from './FileTreeItem'

export const FileTree = ({ rootPath = '/' }: { rootPath?: string }) => {
  const { files, isLoading } = useDirectory(rootPath)

  if (isLoading) return <Skeleton />

  return (
    <Tree>
      {files.map(file => (
        <FileTreeItem key={file.path} file={file} />
      ))}
    </Tree>
  )
}
```

## File Structure

```
ide-ui/
├── FileTree/
│   ├── FileTree.tsx
│   ├── FileTreeItem.tsx
│   ├── FileTreeSkeleton.tsx
│   ├── useFileTree.ts        # component-specific hook
│   └── index.ts             # public API
│
├── CodeEditor/
│   ├── CodeEditor.tsx
│   ├── CodeEditorSkeleton.tsx
│   ├── useCodeEditor.ts
│   └── index.ts
│
├── Tabs/
│   ├── Tabs.tsx
│   ├── Tab.tsx
│   ├── TabList.tsx
│   ├── TabPanel.tsx
│   ├── useTabs.ts
│   └── index.ts
│
├── Breadcrumb/
│   ├── Breadcrumb.tsx
│   ├── BreadcrumbItem.tsx
│   ├── BreadcrumbSeparator.tsx
│   └── index.ts
│
├── FileExplorer/
│   ├── FileExplorer.tsx
│   ├── FileExplorerToolbar.tsx
│   ├── FileExplorerContextMenu.tsx
│   └── index.ts
│
├── ContextMenu/
│   ├── ContextMenu.tsx
│   ├── ContextMenuItem.tsx
│   ├── ContextMenuSeparator.tsx
│   └── index.ts
│
├── StatusBar/
│   ├── StatusBar.tsx
│   ├── StatusBarItem.tsx
│   ├── StatusBarCursor.tsx
│   └── index.ts
│
├── SearchBar/
│   ├── SearchBar.tsx
│   ├── SearchResults.tsx
│   └── index.ts
│
├── hooks/
│   ├── useVFS.ts              # Core hook - required by most components
│   ├── useFile.ts
│   ├── useDirectory.ts
│   ├── useWatcher.ts
│   ├── useLock.ts
│   └── useTabs.ts
│
├── lib/
│   ├── utils.ts              # cn(), clsx helpers
│   └── vfs.ts                # VFS instance singleton
│
├── index.ts                  # Re-exports everything
├── index.css                 # Base styles + CSS variables
├── components.css            # Component styles
└── package.json
```

### Component Structure Pattern

Each component follows the same pattern:

```
ComponentName/
├── ComponentName.tsx         # Main component
├── ComponentNameSkeleton.tsx # Loading state
├── ComponentNameProps.ts     # TypeScript interfaces
├── useComponentName.ts      # Custom hook (optional)
├── sub-components/          # Internal parts (optional)
│   ├── ComponentNameItem.tsx
│   └── ComponentNameGroup.tsx
├── index.ts                  # Barrel export: export { ComponentName }
└── ComponentName.css         # Component-specific styles
```

### index.ts Pattern

```ts
// FileTree/index.ts
export { FileTree } from './FileTree'
export type { FileTreeProps } from './FileTreeProps'
export { FileTreeSkeleton } from './FileTreeSkeleton'
```

### Usage Examples

```ts
// Minimal usage - just drop in
import { FileTree } from '@deessejs/ide-ui/FileTree'

<FileTree rootPath="/src" />

// With customization - props-based
import { FileTree } from '@deessejs/ide-ui/FileTree'

<FileTree
  rootPath="/src"
  onFileClick={(path) => openTab(path)}
  onFileContextMenu={(path) => showMenu(path)}
  showIcons={true}
  defaultExpanded={['/src/components']}
/>

// Compose components together
import { FileTree } from '@deessejs/ide-ui/FileTree'
import { Breadcrumb } from '@deessejs/ide-ui/Breadcrumb'
import { Tabs } from '@deessejs/ide-ui/Tabs'

<div className="ide-layout">
  <aside className="sidebar">
    <Breadcrumb path={currentPath} onNavigate={navigate} />
    <FileTree rootPath={currentPath} onFileClick={openFile} />
  </aside>
  <main className="editor">
    <Tabs tabs={openTabs} onClose={closeTab} />
    <CodeEditor file={activeFile} />
  </main>
</div>
```

## Dependencies

Each component only imports what it needs.

| Package | Purpose |
|---------|---------|
| `react` | UI framework |
| `clsx` | Class name utility (used in lib/utils.ts) |
| `@deessejs/maybe` | Maybe monad (optional hook returns) |
| `@deessejs/result` | Result monad (optional hook returns) |

## VFS Instance

The VFS instance must be provided. There are two ways:

### Option 1: Context Provider

```ts
// Wrap your app with the provider
import { VFSProvider } from '@deessejs/ide-ui'

const vfs = createFileSystem()

<VFSProvider vfs={vfs}>
  <App />
</VFSProvider>

// Components auto-connect via context
import { FileTree } from '@deessejs/ide-ui/FileTree'
// FileTree automatically gets vfs from context
```

### Option 2: Singleton (for simple cases)

```ts
// lib/vfs.ts - create once, reuse everywhere
import { createFileSystem } from '@deessejs/vfs'

export const vfs = createFileSystem({
  root: folder({ name: '/', children: {} })
})

// Components check for singleton if no context
import { getVFS } from '@deessejs/ide-ui/lib/vfs'
```

## Hooks Integration

Hooks work with both approaches:

```ts
// useVFS returns VFS from context or singleton
import { useVFS } from '@deessejs/ide-ui/hooks/useVFS'

const MyComponent = () => {
  const vfs = useVFS()
  // Works whether provided via context or singleton
}
```

## Related Documentation

- [VFS Module](../packages/vfs/README.md)
- [IDE Core Architecture](./architecture.md)
- [Shell Integration](./shell.md)
