# Collapsible Sidebar Sections

**Date:** 2026-03-13
**Status:** Approved

## Summary

Add the ability to collapse and expand category sections in the NoDash sidebar. Users click a category header to toggle its methods list open or closed.

## Decisions

- **Toggle indicator:** `+` (collapsed) / `−` (expanded) on the right side of each category header
- **Default state:** All categories expanded on page load (same as current behavior)
- **Approach:** Vue reactive `Set` — no localStorage persistence
- **Category header element:** Keep as `<div>` with `@click` (avoids button reset styles, consistent with existing code)
- **Search interaction:** No special handling needed. When a search query is active, the Sidebar receives only matching `filteredMethods`, so categories with no matches don't render at all — there is no risk of a user missing results inside a collapsed section. Collapsed state is not reset on search.

## Design

### Sidebar component — migrate to Composition API (`index.html`, `Sidebar` component ~line 3149)

The existing component uses Options API (`computed:`). Move it fully to `setup()` for consistency when adding state:

```js
const Sidebar = defineComponent({
  props: {
    methods:        { type: Array,  required: true },
    activeMethodId: { type: String, default: '' },
  },
  setup(props) {
    const collapsedCategories = ref(new Set());

    const categories = computed(() => {
      const order = ['Array','Collection','Function','Lang','Object','String'];
      const map = {};
      for (const m of props.methods) {
        if (!map[m.category]) map[m.category] = [];
        map[m.category].push(m);
      }
      return order.filter(cat => map[cat]).map(cat => ({ name: cat, methods: map[cat] }));
    });

    function toggleCategory(name) {
      const s = new Set(collapsedCategories.value);
      s.has(name) ? s.delete(name) : s.add(name);
      collapsedCategories.value = s;
    }

    return { categories, collapsedCategories, toggleCategory };
  },
  // template below
});
```

A new `Set` is constructed on each toggle so Vue detects the ref change.

### Template

```html
<nav class="sidebar">
  <template v-for="cat in categories" :key="cat.name">
    <div class="sidebar-category" @click="toggleCategory(cat.name)">
      {{ cat.name }}
      <span class="sidebar-toggle">{{ collapsedCategories.has(cat.name) ? '+' : '−' }}</span>
    </div>
    <a
      v-for="m in cat.methods"
      v-show="!collapsedCategories.has(cat.name)"
      :key="m.id"
      :href="'#' + m.id"
      :class="['sidebar-method', { active: m.id === activeMethodId }]"
    >{{ m.name }}</a>
  </template>
</nav>
```

Note: `collapsedCategories` is a top-level ref returned from `setup()`, so Vue 3 auto-unwraps it in the template — `.has()` is called on the inner `Set` directly, no `.value` needed in the template.

### CSS — edit the existing `.sidebar-category` rule in place

The existing rule (lines 47–51) gains three new properties. Edit the single block:

```css
.sidebar-category {
  padding: 6px 16px 2px; font-size: 11px; font-weight: 700;
  text-transform: uppercase; letter-spacing: 0.08em;
  color: #6b7280; margin-top: 8px;
  display: flex; justify-content: space-between; align-items: center;
  cursor: pointer; user-select: none;
}
```

Add a new rule for the toggle glyph:

```css
.sidebar-toggle {
  font-size: 14px;
  font-weight: 400;
  line-height: 1;
  color: #9ca3af;
}
```

## Behavior

- All categories expanded on load
- Clicking a category header `<div>` toggles collapsed/expanded
- `+` shown when collapsed, `−` when expanded
- The active method highlight is unaffected by collapse state
- No auto-expand when active method changes — user's manual state is the truth
- State resets on page reload (no persistence)
- No keyboard toggle (div, not button) — acceptable for a reference site

## Scope

Single file change: `index.html`. No new files, no dependencies.
