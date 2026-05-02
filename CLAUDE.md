# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Agent Instructions

For detailed architectural guidance, build commands, and development workflows, see **[AGENTS.md](./AGENTS.md)**.

---

## Quick Reference

### Repository Overview

Lexical (v0.40.0) is an extensible text editor framework by Meta, built as a TypeScript monorepo. It targets the browser and supports React, headless (server-side), and plain-DOM usage. The package manager is **pnpm** (>=9.0.0); Node.js 18–22 is required.

### Most-Used Commands

```bash
# Install dependencies
pnpm install

# Build
pnpm run build              # dev mode
pnpm run build-prod         # clean + production
pnpm run build-types        # TypeScript type declarations + validation

# Test
pnpm run test-unit          # Vitest unit tests (no watch)
pnpm run test-e2e-chromium  # Playwright E2E (needs server running first)

# Dev servers
pnpm run dev                # playground only (http://localhost:3000)
pnpm run start              # playground + collab server
pnpm run start:website      # Docusaurus docs (http://localhost:3001)

# Code quality (all run in CI)
pnpm run lint               # ESLint
pnpm run prettier           # check formatting
pnpm run prettier:fix       # auto-fix formatting
pnpm run flow               # Flow type check
pnpm run tsc                # TypeScript check
pnpm run ci-check           # tsc + flow + prettier + lint in parallel
```

---

## Codebase Structure

### Monorepo Layout

```
lexical/
├── packages/
│   ├── lexical/                  # Core framework
│   ├── lexical-react/            # React bindings & plugins
│   ├── lexical-headless/         # Headless/server-side editor
│   ├── lexical-rich-text/        # Rich text (headings, quotes)
│   ├── lexical-plain-text/       # Plain text editing
│   ├── lexical-list/             # List nodes (ol/ul/checklist)
│   ├── lexical-table/            # Table support
│   ├── lexical-code/             # Code blocks + syntax highlight
│   ├── lexical-code-shiki/       # Shiki-based highlighting
│   ├── lexical-link/             # Link nodes & utilities
│   ├── lexical-markdown/         # Markdown import/export
│   ├── lexical-html/             # HTML serialization
│   ├── lexical-history/          # Undo/redo
│   ├── lexical-yjs/              # Real-time collab via Yjs
│   ├── lexical-selection/        # Selection helpers
│   ├── lexical-utils/            # General utilities
│   ├── lexical-clipboard/        # Clipboard support
│   ├── lexical-hashtag/          # Hashtag nodes
│   ├── lexical-overflow/         # Overflow node
│   ├── lexical-mark/             # Mark/annotation nodes
│   ├── lexical-offset/           # Offset utilities
│   ├── lexical-text/             # Text helpers
│   ├── lexical-dragon/           # Dragon NaturallySpeaking support
│   ├── lexical-tailwind/         # Tailwind CSS integration
│   ├── lexical-eslint-plugin/    # Public ESLint rules
│   ├── lexical-eslint-plugin-internal/ # Internal ESLint rules
│   ├── lexical-devtools/         # Browser DevTools extension
│   ├── lexical-devtools-core/    # DevTools shared core
│   ├── lexical-extension/        # Extension utilities (browser)
│   ├── lexical-playground/       # Full-featured demo app (Vite)
│   ├── lexical-website/          # Docusaurus documentation site
│   └── shared/                   # Internal shared utilities
├── scripts/                      # Build, release, and tooling scripts
├── flow-typed/                   # Flow type definitions
├── babel.config.js
├── jest.config.js
├── playwright.config.js
├── vitest.config.mts
├── tsconfig.json / tsconfig.build.json / tsconfig.test.json
└── pnpm-workspace.yaml
```

### Core Package (`packages/lexical/src/`)

Key source files:

| File | Purpose |
|------|---------|
| `LexicalEditor.ts` | `LexicalEditor` class — the top-level editor instance |
| `LexicalEditorState.ts` | Immutable `EditorState` (node tree + selection) |
| `LexicalNode.ts` | Base `LexicalNode` class all nodes extend |
| `LexicalUpdates.ts` | Update scheduling, batching, reconciliation triggering |
| `LexicalReconciler.ts` | DOM reconciliation (diff & patch) |
| `LexicalSelection.ts` | Selection types and helpers |
| `LexicalCommands.ts` | Built-in command definitions |
| `LexicalEvents.ts` | DOM event handlers wired to the editor |
| `LexicalNodeState.ts` | `createState` / `$setState` / `$getState` API |
| `LexicalGC.ts` | Garbage collection of orphaned nodes |
| `nodes/` | Built-in node types (Text, Element, Root, Paragraph, LineBreak, Tab, Decorator) |
| `caret/` | Caret navigation utilities (`LexicalCaret.ts`, `LexicalCaretUtils.ts`) |
| `extension-core/` | `defineExtension` — **experimental** extension system |

---

## Architecture & Key Concepts

### Core Data Model

- **`LexicalEditor`** — created via `createEditor()`. Owns the EditorState, DOM root, registered nodes/commands/transforms/listeners.
- **`EditorState`** — immutable snapshot: a node tree rooted at `RootNode` plus the current selection. Fully serializable to/from JSON.
- **Double-buffering** — on `editor.update()`, the current EditorState is cloned as a mutable work-in-progress. After all synchronous mutations, the reconciler diffs and patches the DOM, then the new state becomes the immutable current state.

### `$` Function Convention

Functions prefixed with `$` (e.g., `$getRoot()`, `$getSelection()`, `$createParagraphNode()`) **must only be called inside**:

- `editor.update(() => { ... })` — for mutations
- `editor.read(() => { ... })` — for read-only access
- Node transforms (have implicit update context)
- Command handlers (have implicit update context)

Calling them outside these contexts throws in development. This is analogous to React hooks but enforces synchronous context rather than call order.

### Node Hierarchy

```
RootNode
└── ElementNode (ParagraphNode, HeadingNode, ListNode, …)
    └── TextNode | DecoratorNode | LineBreakNode | TabNode
```

- **`TextNode`** — leaf text with formatting bitmask (bold, italic, underline, etc.)
- **`ElementNode`** — container that holds children
- **`DecoratorNode`** — placeholder for non-Lexical content (React components, embeds)
- All node classes must implement `getType()`, `clone()`, `importJSON()`, `exportJSON()`

### Node Immutability & Keys

Nodes are frozen after reconciliation. To mutate, call `node.getWritable()` which clones the node into the work-in-progress state. The stable **key** (`node.__key`) is preserved across mutations so `NodeMap` lookups always find the latest version.

### Command System

```ts
const MY_COMMAND = createCommand<Payload>('MY_COMMAND');
editor.dispatchCommand(MY_COMMAND, payload);
editor.registerCommand(MY_COMMAND, (payload) => {
  // return true to stop propagation
  return false;
}, COMMAND_PRIORITY_EDITOR);
```

Commands propagate through handlers in priority order (LOW → EDITOR); the first handler that returns `true` stops propagation.

### Node Transforms

```ts
editor.registerNodeTransform(TextNode, (node) => {
  // called automatically when TextNodes change during an update
  // has implicit update context — $ functions are safe here
});
```

Transforms are called during reconciliation for any dirty nodes of the registered type.

### React Plugin Pattern

```tsx
function MyPlugin() {
  const [editor] = useLexicalComposerContext();
  useEffect(() => {
    return editor.registerUpdateListener(({editorState}) => {
      editorState.read(() => { /* read state */ });
    });
  }, [editor]);
  return null;  // plugins render nothing
}
```

All `editor.register*()` calls return a cleanup/unsubscribe function.

### NodeState API (new in recent versions)

```ts
// Define typed state outside a node class
const fooState = createState('foo', { parse: (v) => typeof v === 'string' ? v : '' });

class MyNode extends TextNode {
  getFoo(): string { return $getState(this, fooState); }
  setFoo(v: string): this { return $setState(this, fooState, v); }
}
```

`createState` / `$getState` / `$setState` decouple arbitrary typed metadata from the node's own class fields.

### Extension System (`defineExtension`) — Experimental

```ts
export const MyExtension = defineExtension({
  name: 'my',
  nodes: [MyNode],
  config: safeCast<MyConfig>({ defaultValue: 1 }),
});
```

Extensions encapsulate nodes, config, and optional peer dependencies for reusable editor features. The API is marked `@experimental`.

---

## Type System

The codebase uses **both TypeScript and Flow**:

- Source is TypeScript (`.ts`, `.tsx`); Flow definitions are generated under `packages/*/flow/`.
- Both type systems are checked in CI via `pnpm run ci-check`.
- When adding or modifying public APIs, update types in both systems.
- `pnpm run flow` — check Flow; `pnpm run tsc` — check TypeScript.

---

## Testing

### Unit Tests (Vitest)

- Location: `packages/**/__tests__/unit/**/*.test.{ts,tsx}`
- Run: `pnpm run test-unit` (or `pnpm run test-unit-watch` for watch mode)
- Debug: `pnpm run debug-test-unit` (opens Node inspector)
- Test environment: jsdom with React; `__DEV__` is `true`

### E2E Tests (Playwright)

- Location: `packages/lexical-playground/__tests__/e2e/**/*.spec.{ts,mjs}`
- Requires the playground dev server to be running first:
  ```bash
  pnpm run dev          # terminal 1
  pnpm run test-e2e-chromium  # terminal 2
  ```
- Debug: `pnpm run debug-test-e2e-chromium` (headed browser + Playwright inspector)
- E2E modes: `plain-text`, `rich-text` (default), `rich-text-with-collab`, `rich-text-with-collab-v2`
- Flaky tests are tagged `@flaky` and excluded in CI runs

---

## Code Conventions

### File Headers

Every source file must start with:

```ts
/**
 * Copyright (c) Meta Platforms, Inc. and affiliates.
 *
 * This source code is licensed under the MIT license found in the
 * LICENSE file in the root directory of this source tree.
 *
 */
```

### Naming

- `$functionName` — can only be called inside `editor.update()` / `editor.read()` / transforms / command handlers
- `$createXxxNode()` — factory for a node type (follows `$` convention)
- `LexicalXxx.ts` — core framework files in `packages/lexical/src/`
- Plugin files in `@lexical/react`: `LexicalXxxPlugin.tsx`

### Imports

ESLint (`simple-import-sort`) enforces sorted imports. Run `pnpm run prettier:fix` to auto-fix.

### Pre-commit Hook

Husky runs `lint-staged` on commit. Ensure `pnpm run lint` and `pnpm run prettier` pass before committing.

---

## Build System

- **Rollup** bundles each package (`scripts/build.js`)
- Build modes: `development` (default), `production` (`--prod`), `www` (Meta internal `--www`), `release` (`--codes`)
- Output: CommonJS + ESM per package
- TypeScript source → compiled declarations via `tsconfig.build.json`
- Package metadata managed by `scripts/shared/packagesManager.js`

### Adding a New Package

After creating `packages/lexical-<name>/`:

```bash
pnpm run update-packages   # updates version, tsconfig, flowconfig, docs, www stubs
```

---

## Development Workflow

1. `pnpm install` — install all workspace dependencies
2. Make changes in the relevant package(s)
3. `pnpm run build` — rebuild (required for cross-package changes to be picked up)
4. `pnpm run test-unit` — run unit tests
5. `pnpm run ci-check` — run all linting and type checks before committing
6. For UI/playground work: `pnpm run dev` + verify in browser at http://localhost:3000

### Custom Node Checklist

When creating a custom node:

1. Extend `TextNode`, `ElementNode`, or `DecoratorNode`
2. Implement `static getType(): string`
3. Implement `static clone(node: YourNode): YourNode`
4. Implement `static importJSON(json): YourNode`
5. Implement `exportJSON(): SerializedYourNode`
6. Implement `createDOM()` / `updateDOM()` (for element/decorator nodes)
7. Register in editor config: `nodes: [YourCustomNode]`
8. Export a `$createYourNode()` factory (follows `$` convention)

---

## Important Gotchas

- **Never store node references across update boundaries** — nodes are versioned; always re-fetch via key within a read/update context.
- **`editor.read()` flushes pending updates** before giving you a consistent state. `editor.getEditorState().read()` uses the last reconciled state without flushing.
- **Inside `editor.update()`**, you see the pending (unreconciled) state — transforms and reconciliation haven't run yet.
- **Nested updates are allowed** (`editor.update(() => editor.update(...))`) but nesting reads inside updates (or vice versa) is not, except that a read at the end of an update flushes it.
- **E2E tests are not unit tests** — the playground server must be running and the test targets real browser behavior, not jsdom.
