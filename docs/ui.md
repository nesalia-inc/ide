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

Use CSS variables for theming:

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

## File Structure

```
ide-ui/
├── src/
│   ├── components/
│   │   ├── FileTree/
│   │   │   ├── FileTree.tsx
│   │   │   ├── FileTreeItem.tsx
│   │   │   └── index.ts
│   │   ├── FileExplorer/
│   │   ├── CodeEditor/
│   │   ├── Tabs/
│   │   ├── Breadcrumb/
│   │   ├── ContextMenu/
│   │   ├── SearchBar/
│   │   └── StatusBar/
│   ├── hooks/
│   │   ├── useVFS.ts
│   │   ├── useFile.ts
│   │   ├── useDirectory.ts
│   │   ├── useWatcher.ts
│   │   ├── useLock.ts
│   │   ├── useTabs.ts
│   │   └── useSearch.ts
│   ├── store/
│   │   └── ideStore.ts
│   ├── index.ts
│   └── index.css
├── package.json
└── tsconfig.json
```

## Dependencies

| Package | Purpose |
|---------|---------|
| `react` | UI framework |
| `zustand` | State management |
| `@monaco-editor/react` | Code editor |
| `@tanstack/react-virtual` | Virtualization |
| `clsx` | Class name utility |

## Related Documentation

- [VFS Module](../packages/vfs/README.md)
- [IDE Core Architecture](./architecture.md)
- [Shell Integration](./shell.md)
