# NoDash — Design Spec

**Date:** 2026-03-13

## Overview

NoDash is a Vue 3 documentation site styled after [lodash.com/docs](https://lodash.com/docs/4.17.23), but showcasing native JavaScript alternatives to Lodash methods sourced from the [You Don't Need Lodash/Underscore](https://github.com/you-dont-need/You-Dont-Need-Lodash-Underscore) repository. The goal is a clean, fast, searchable reference for developers who want to drop their Lodash dependency.

---

## Tech Stack

- **Framework:** Vue 3 — loaded via CDN using an ES module importmap (no bundler, no npm, no build step)
- **Entry point:** A single `index.html` file; open with any static file server or `python3 -m http.server`
- **Language:** Plain JavaScript (no TypeScript)
- **Components:** Defined as plain `.js` files using `Vue.defineComponent` with `template` strings — no `.vue` SFC files
- **Routing:** Hash-based anchor navigation only — no Vue Router. Sidebar clicks update the hash via native `<a href>` behavior; scrolling does NOT update the hash.
- **Styling:** A single `styles.css` file loaded in `index.html`; `html { scroll-behavior: smooth; }`
- **No build tools, no npm, no dark mode toggle** — code blocks use a dark background for readability; the page is always light

---

## File Structure

```
index.html             # entry: importmap, styles, mount point, loads app.js
styles.css             # all CSS
src/
  app.js               # root component: layout shell, shared state
  data/
    methods.js         # exported array of ~96 method objects
  components/
    TopBar.js          # fixed header: logo, search input, tagline badge
    Sidebar.js         # fixed left nav: category headings + method anchor links
    MethodList.js      # scrolling content + IntersectionObserver
    MethodCard.js      # single method display
    CodeBlock.js       # syntax-highlighted code + copy button
    CompatBadges.js    # browser support badge row
```

**`index.html` structure:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>NoDash</title>
  <link rel="stylesheet" href="styles.css" />
  <script type="importmap">
    {
      "imports": {
        "vue": "https://unpkg.com/vue@3/dist/vue.esm-browser.prod.js"
      }
    }
  </script>
</head>
<body>
  <div id="app"></div>
  <script type="module" src="src/app.js"></script>
</body>
</html>
```

Each `.js` component file uses ES module syntax:

```js
import { defineComponent, ref, computed } from 'vue';
import Sidebar from './components/Sidebar.js';

export default defineComponent({
  components: { Sidebar },
  template: `<div>...</div>`,
  setup() { ... }
});
```

`src/app.js` also calls `createApp` and mounts:

```js
import { createApp } from 'vue';
import App from './app.js'; // the root component is defined in the same file

createApp(App).mount('#app');
```

> Note: `app.js` exports the root component definition AND calls `createApp`/`mount` at the bottom. All child components are imported from `src/components/`.

---

## Content Strategy

All method data lives in `src/data/methods.js` as an exported array of method objects.

**Method object shape:**

```js
{
  id: 'chunk',               // string — unique, lowercase, no spaces. Used as DOM id and URL hash.
  name: '_.chunk',           // string — lodash method name shown in heading
  signature: '_.chunk(array, size)', // string — full signature for h2
  category: 'Array',         // string — exactly one of the 6 values below

  description: 'Creates an array of elements split into groups...', // string

  // args: required field; empty array [] for zero-argument methods
  args: [
    { name: 'array', type: 'Array',  description: 'The array to process' },
    { name: 'size',  type: 'number', description: 'The length of each chunk' },
  ],
  // arg shape: { name: string, type: string, description: string }

  // returns: null for void methods; object otherwise
  returns: { type: 'Array', description: 'the new array of chunks' },

  // examples: at least one entry required; label is 'ES6' or 'ES5'
  // Content is native-only (not lodash vs. native comparison)
  examples: [
    { label: 'ES6', code: `const chunk = (arr, size) =>
  Array.from({ length: Math.ceil(arr.length / size) },
    (_, i) => arr.slice(i * size, i * size + size));

chunk([1, 2, 3, 4], 2);
// => [[1, 2], [3, 4]]` },
  ],
  // Some methods have both an ES6 and ES5 example entry

  // compat: exactly 5 entries, always in this order: Chrome, Firefox, Safari, Edge, IE
  // supported: boolean — canonical flag; drives badge color
  // version: human-readable string (e.g. '45+') when supported is true; null when false
  // Badge text: "${browser} ${version ?? '✗'}" — e.g. "Chrome 45+", "IE ✗"
  compat: [
    { browser: 'Chrome',  version: '45+', supported: true  },
    { browser: 'Firefox', version: '32+', supported: true  },
    { browser: 'Safari',  version: '9+',  supported: true  },
    { browser: 'Edge',    version: '12+', supported: true  },
    { browser: 'IE',      version: null,  supported: false },
  ],
}
```

**6 categories**, ~96 methods total:
- **Array** — chunk, compact, concat, difference, drop, fill, filter, find, findIndex, flatten, head, indexOf, join, last, reverse, slice, sortBy, without, zip, etc.
- **Collection** — each, every, filter, find, map, reduce, size, some, sortBy
- **Function** — after, bind, debounce, memoize, partial, throttle
- **Lang** — clone, isEmpty, isDate, isFunction, isNaN, isNull, isNumber, isObject, isString, isUndefined
- **Object** — assign, defaults, entries, fromEntries, keys, omit, pick, values
- **String** — capitalize, endsWith, padStart, padEnd, repeat, replace, split, startsWith, toLower, toUpper, trim, trimEnd, trimStart

---

## Architecture & Data Flow

**`src/app.js` (root component) holds:**
- `searchQuery` ref — string, default `''`; bound via `v-model` to `TopBar`
- `activeMethodId` ref — string; updated by `update:activeMethodId` event from `MethodList` AND by the `hashchange` window event listener
- `filteredMethods` computed (see Search section)

**App template:**

```html
<TopBar v-model="searchQuery" />
<div class="layout">
  <Sidebar :methods="filteredMethods" :activeMethodId="activeMethodId" />
  <div class="content-wrap">
    <MethodList
      :methods="filteredMethods"
      :searchQuery="searchQuery"
      @update:activeMethodId="activeMethodId = $event"
    />
    <footer>...</footer>
  </div>
</div>
```

**On mount:**
1. Read `window.location.hash.slice(1)`
2. If non-empty and matches a valid method id: set `activeMethodId = hash`; browser's native anchor scroll handles positioning
3. If no hash: set `activeMethodId` to the `id` of the first method in the full list
4. Add `hashchange` listener on `window`: on each hash change, if new hash matches a valid method id, immediately set `activeMethodId`. This provides instant sidebar highlight on click without waiting for IntersectionObserver.
5. Remove the `hashchange` listener on unmount

**When `filteredMethods` changes:** if `activeMethodId` is no longer in `filteredMethods`, reset to the first entry's id (or `''` if empty). When `activeMethodId` is `''`, Sidebar renders no active highlight.

---

## Component Responsibilities

### `TopBar.js`
- Props: `modelValue` (string)
- Emits: `update:modelValue` — enables `v-model` in parent
- Template: logo `"NoDash"` (with `"Dash"` in accent color `#a5b4fc` on the indigo bar), search `<input>` emitting `update:modelValue` on every `input` event, static badge `"You Don't Need Lodash"`

### `Sidebar.js`
- Props: `methods` (filtered array), `activeMethodId` (string)
- Groups methods by `category` using a computed; categories with zero methods are omitted
- Template: category headings + `<a :href="'#' + method.id">` anchor links
- **No `scrollIntoView` calls** — native anchor nav + `scroll-behavior: smooth` handles scrolling
- Applies `.active` class to the link where `method.id === activeMethodId`; no active class when `activeMethodId === ''`

### `MethodList.js`
- Props: `methods` (filtered array), `searchQuery` (string)
- Emits: `update:activeMethodId` (string)
- Root element: `<main>` (styled with `padding: 32px 40px; max-width: 900px; flex: 1`)
- If `methods.length === 0`: renders `<p class="empty-state">No methods found for '{{ searchQuery }}'</p>`
- Otherwise: renders one `MethodCard` per method
- **IntersectionObserver:**
  - `threshold: [0, 0.1, 0.3]` — multiple thresholds handle short cards
  - A `watch` on the `methods` prop (with `{ flush: 'post' }`) disconnects any existing observer, then creates a new one and observes all current `<section>` card elements. This covers both initial mount and subsequent prop changes.
  - Disconnects on unmount
  - Callback: collect `isIntersecting: true` entries. If none, do nothing. Otherwise pick the entry with `boundingClientRect.top ≥ 0` closest to 0 (topmost visible); if all have `top < 0`, pick the largest (least negative) top. Emit `update:activeMethodId` with that entry's element's `dataset.methodId`.

### `MethodCard.js`
- Props: `method` (single method object)
- Root element: `<section :id="method.id" :data-method-id="method.id">`
- Template:
  - `<h2>{{ method.signature }}</h2>`
  - Subtitle: `"Native (ES6+)"` / `"Native (ES5+)"` / `"Native"` based on `examples[0].label`
  - Description paragraph
  - Args table (columns: Name, Type, Description) — only if `args.length > 0`
  - Returns line — only if `returns !== null`
  - `<CodeBlock>` for each example
  - `<CompatBadges>` with `method.compat`

### `CodeBlock.js`
- Props: `code` (string), `label` (string)
- Template: label text `"Example ({{ label }})"` above a dark `<pre>` block
- `<pre>` styles: `background: #1e1e2e; color: #cdd6f4; border-radius: 6px; padding: 16px 20px; font-size: 13px; overflow-x: auto; position: relative`
- **Syntax highlighting:** rendered via `v-html` on the `<pre>`. The `code` string is first HTML-escaped (`&`, `<`, `>`, `"`) then processed by regex replacements in this order (to prevent re-highlighting):
  1. Line comments: `//.*$` (multiline) → `<span class="cm">$&</span>`
  2. String literals: single/double/template-quoted → `<span class="str">$&</span>`
  3. Numeric literals: `\b\d+\.?\d*\b` → `<span class="num">$&</span>`
  4. Keywords: `\b(const|let|var|return|function|new|if|else|for|of|in|typeof)\b` → `<span class="kw">$&</span>`
- CSS classes: `.kw { color: #cba6f7 }`, `.str { color: #a6e3a1 }`, `.num { color: #fab387 }`, `.cm { color: #6c7086 }`
- Copy button: `position: absolute; top: 8px; right: 8px`. Calls `navigator.clipboard.writeText(this.code)`. Success: shows `"Copied!"` for 1500ms then reverts to `"Copy"`. Failure: silently ignored.
- Since `v-html` is used, the copy button is rendered separately as a real DOM element (not inside the `v-html` content), overlaid via `position: absolute` on the `<pre>`.

### `CompatBadges.js`
- Props: `compat` (array of exactly 5 entries)
- For each entry renders a `<span>` badge:
  - Text: `"${entry.browser} ${entry.version ?? '✗'}"` — e.g. `"Chrome 45+"`, `"IE ✗"`
  - Color driven by `entry.supported`: green (`#f0fdf4` bg, `#15803d` text, `#bbf7d0` border) when `true`; amber (`#fffbeb` bg, `#b45309` text, `#fde68a` border) when `false`

---

## Search Behavior

`filteredMethods` computed in `src/app.js`:
- Lowercase the `searchQuery`, then include a method if **any** of (OR logic):
  1. `method.id.toLowerCase().includes(query)`
  2. `method.name.toLowerCase().includes(query)`
  3. `method.category.toLowerCase() === query` (exact match — avoids substring false positives on enum values)
- Empty `searchQuery` returns full array
- Both `Sidebar` and `MethodList` receive the same `filteredMethods`

---

## Layout & Visual Design

**Minimum width: 768px. No mobile/responsive support.**

| Element | Style |
|---|---|
| `html` | `scroll-behavior: smooth` |
| Top bar | `position: fixed; top: 0; left: 0; right: 0; height: 48px; background: #3f51b5; z-index: 100; box-shadow: 0 1px 4px rgba(0,0,0,0.3)` |
| Sidebar | `position: fixed; top: 48px; bottom: 0; width: 200px; background: #fafafa; border-right: 1px solid #e5e7eb; overflow-y: auto; padding: 16px 0` |
| `.layout` | `display: flex; padding-top: 48px` |
| `.content-wrap` | `margin-left: 200px; display: flex; flex-direction: column; min-height: calc(100vh - 48px)` |
| `main` (MethodList root) | `padding: 32px 40px; max-width: 900px; flex: 1` (left-aligned, no centering) |
| Method `<section>` | `margin-bottom: 40px; padding-bottom: 40px; border-bottom: 1px solid #f0f0f0` |
| `footer` | `border-top: 1px solid #e5e7eb; background: #fafafa; padding: 20px 40px; display: flex; justify-content: space-between; font-size: 12px; color: #9ca3af` — static, inside `.content-wrap` below `<main>` |

---

## Footer

```html
<footer>
  <span>Based on <a href="https://github.com/you-dont-need/You-Dont-Need-Lodash-Underscore">You Don't Need Lodash/Underscore</a></span>
  <span>Built with <a href="https://claude.ai/code">Claude Code</a></span>
</footer>
```

---

## Verification

1. `python3 -m http.server` (or any static server) in the project root; open `http://localhost:8000` → all ~96 method cards render with no console errors
2. Search `"chunk"` → only the chunk card in main; sidebar shows only "Array" category with only "chunk"
3. Search `"zzz"` → empty state message `"No methods found for 'zzz'"`; sidebar empty
4. Clear search → all 96 methods and 6 categories restore
5. Search `"Array"` → only Array-category methods (exact category match; no spurious results)
6. Click `"compact"` in sidebar → page scrolls smoothly to `#compact`; sidebar "compact" link has `.active` class
7. Scroll manually past several cards → sidebar active highlight updates to the topmost visible card
8. Open `http://localhost:8000/#chunk` directly → page scrolls to chunk card on load; sidebar highlights "chunk"
9. Copy button → `"Copied!"` flash; clipboard contains raw code
10. Compat badges on `compact` → five green badges; on `chunk` → four green + one amber `"IE ✗"`
11. Footer links → both open correctly
