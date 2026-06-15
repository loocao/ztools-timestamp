# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

- **Type**: ZTools Plugin (timestamp conversion tool)
- **Stack**: Vue 3 (Composition API, `<script setup lang="ts">`), Vite 6, TypeScript 5, Tailwind CSS v4
- **Note**: This is a plugin for the ZTools desktop app — it requires the ZTools runtime to function. Dev mode serves from `http://localhost:5173`.

## Build Commands

```bash
npm install     # Install dependencies
npm run dev     # Start dev server (http://localhost:5173)
npm run build   # Production build (runs vue-tsc first, outputs to dist/)
```

No test framework or linter is configured.

## Architecture

### Plugin Routing
The app uses a simple string-based router in `App.vue`. There are no external router libraries — route state is a single `ref<string>`:

1. `window.ztools.onPluginEnter(param)` sets `route.value = param.code`
2. `App.vue` conditionally renders components based on `route`
3. Each feature maps to one Vue component registered in `App.vue`

### Component Structure
```
src/
├── App.vue                         # Root: route state + conditional rendering
├── main.ts                         # Vue app entry
├── env.d.ts                        # Type declarations (Window.services, Vite, *.vue modules)
├── assets/css/main.css             # Tailwind import + base styles (@import 'tailwindcss')
└── components/
    └── timestamp/
        └── timestamp.vue           # The only feature component (timestamp conversion)
```

### ZTools API (`window.ztools`)
Provided by `@ztools-center/ztools-api-types`. Key methods:
- `onPluginEnter(callback)` — plugin entry point, receives `{ code, payload, ... }`
- `onPluginOut(callback)` — plugin exit
- `copyText(text)` — copy to clipboard
- `outPlugin(success)` — exit plugin (called after successful copy)
- `getClipboardContent()` — read clipboard
- `hideMainWindow()` / `showTip(msg)` — UI controls

### Preload Services (`window.services`)
Node.js capabilities declared in `src/env.d.ts` (`readFile`, `writeTextFile`, `writeImageFile`) and served from `public/preload/services.js`. Currently a stub — not yet used in components.

## TypeScript

- `strict: false`, `noImplicitAny: false` — explicit types preferred but avoid excessive `as any`
- `target: ES2020`, `module: ESNext` (bundler resolution)
- Type declarations: `@ztools-center/ztools-api-types` + manual `Window.services` interface in `src/env.d.ts`

## Styling

- Tailwind CSS v4 with `@tailwindcss/vite` plugin (no PostCSS config needed)
- Import: `@import 'tailwindcss'` in `src/assets/css/main.css` (not `src/main.css`)
- Custom base: `@layer base { * { border-color: var(--color-gray-200); } }`
- Use utility classes in templates exclusively

## Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Components | PascalCase | `Timestamp`, `HelloWorld` |
| Files | kebab-case | `timestamp.vue`, `main.ts` |
| Props | camelCase | `launchParam` |
| Variables | camelCase | `route`, `inputValue` |

## Error Handling

Use try-catch for async operations, always log with meaningful context:

```typescript
try {
  await window.ztools.copyText(text)
} catch (error) {
  console.error('复制失败:', error)
}
```

## Key Conversion Logic (timestamp.vue)

All conversion logic lives in the single `timestamp.vue` component. Key patterns:

- **Input parsing**: `parsedDate` computed property detects numeric (timestamp ms/s) vs date-string inputs
- **ISO 8601**: Custom regex parsing preserves UTC offset info — creates Date via `Date.UTC()` when timezone present, via local constructor when absent
- **Timezone display**: `getTimezoneTime(offsetMinutes)` computes each zone time from UTC components (avoids local timezone interference)
- **Copy-to-exit pattern**: `copyToClipboard` calls `window.ztools.copyText()` then `window.ztools.outPlugin(true)` — copy always exits the plugin
- **Keyboard shortcuts**: `⌘/Ctrl + 1-5` maps to the 5 result rows, registered via `window.addEventListener('keydown', ...)` with `metaKey`/`ctrlKey` detection by `navigator.platform`
- **50+ timezones**: Hardcoded array covering UTC-12:00 to UTC+14:00 with offset in minutes

## Adding a New Feature

1. **Component**: Create `src/components/<name>/timestamp.vue` using `<script setup lang="ts">`, accept `launchParam` prop, derive `inputValue` from `payload`
2. **Route**: Import and add to the conditional render in `App.vue`
3. **Config**: Add feature entry in `public/plugin.json` under `features`, matching `code` to the route name

## Key Files

| File | Purpose |
|------|---------|
| `public/plugin.json` | Plugin metadata, features, commands (text + 9 regex triggers), development URL |
| `public/preload/services.js` | Node.js service extensions (stub) |
| `src/App.vue` | Route registration and conditional rendering |
| `src/env.d.ts` | Type declarations for Vite, Vue SFCs, and `window.services` |
| `src/assets/css/main.css` | Tailwind v4 import + base styles |
| `vite.config.js` | Vite + Vue + Tailwind plugins, base path `./` |
