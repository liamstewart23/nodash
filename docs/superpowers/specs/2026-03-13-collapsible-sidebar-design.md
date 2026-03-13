# Collapsible Sidebar Sections

**Date:** 2026-03-13
**Status:** Approved

## Summary

Add the ability to collapse and expand category sections in the NoDash sidebar. Users click a category header to toggle its methods list open or closed.

## Decisions

- **Toggle indicator:** `+` (collapsed) / `−` (expanded) on the right side of each category header
- **Default state:** All categories expanded on page load (same as current behavior)
- **Approach:** Vue reactive `Set` — no localStorage persistence

## Design

### Sidebar component changes (`index.html` — `Sidebar` component, ~line 3146)

**State:**
Add `setup()` with a single reactive ref:
```js
const collapsedCategories = ref(new Set());
```
Initialized empty so all categories start expanded.

**Toggle method:**
```js
function toggleCategory(name) {
  const s = new Set(collapsedCategories.value);
  s.has(name) ? s.delete(name) : s.add(name);
  collapsedCategories.value = s;
}
```
A new Set is created on each toggle to ensure Vue reactivity.

**Template changes:**
- Category header `<div class="sidebar-category">` becomes a `<button>` (or keeps `div` with `@click`)
- Header displays category name on the left and `+` or `−` on the right using a `<span class="sidebar-toggle">`
- Methods list wrapped with `v-show="!collapsedCategories.has(cat.name)"`

### CSS additions

```css
.sidebar-category {
  /* existing styles remain */
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
  user-select: none;
}
.sidebar-toggle {
  font-size: 14px;
  font-weight: 400;
  line-height: 1;
  color: #9ca3af;
}
```

## Behavior

- All categories expanded on load
- Clicking a category header toggles collapsed/expanded
- When collapsed, the `+` icon shows; when expanded, `−` shows
- The active method highlight is unaffected by collapse state
- No auto-expand when the active method changes (user's manual state is the truth)
- State resets on page reload (no persistence)

## Scope

Single file change: `index.html`. No new files, no dependencies.
