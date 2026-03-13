# Collapsible Sidebar Sections Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `+`/`−` toggle buttons to sidebar category headers so users can collapse and expand method lists.

**Architecture:** Single file change to `index.html`. Migrate the `Sidebar` Vue component from Options API to Composition API, adding a reactive `Set` to track collapsed categories. Category headers get a click handler; method links are hidden with `v-show` when their category is collapsed.

**Tech Stack:** Vue 3 (CDN, no build step), plain HTML/CSS in a single `index.html` file.

---

## Chunk 1: Implement collapsible sidebar

### Task 1: Update CSS

**Files:**
- Modify: `index.html:47-51` (`.sidebar-category` rule)

- [ ] **Step 1: Edit the existing `.sidebar-category` CSS rule in place**

Find the block at lines 47–51:

```css
.sidebar-category {
  padding: 6px 16px 2px; font-size: 11px; font-weight: 700;
  text-transform: uppercase; letter-spacing: 0.08em;
  color: #6b7280; margin-top: 8px;
}
```

Replace it with:

```css
.sidebar-category {
  padding: 6px 16px 2px; font-size: 11px; font-weight: 700;
  text-transform: uppercase; letter-spacing: 0.08em;
  color: #6b7280; margin-top: 8px;
  display: flex; justify-content: space-between; align-items: center;
  cursor: pointer; user-select: none;
}
.sidebar-toggle {
  font-size: 14px;
  font-weight: 400;
  line-height: 1;
  color: #9ca3af;
}
```

- [ ] **Step 2: Open `index.html` in a browser and verify no visual regressions**

Open the file directly (`open index.html` or drag to browser). The sidebar should look identical to before — category headers may shift very slightly due to `display: flex`, but content and spacing should be unchanged.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "style: add flex layout and cursor to sidebar category headers"
```

---

### Task 2: Migrate Sidebar component to Composition API and add collapse logic

**Files:**
- Modify: `index.html:3149-3175` (`Sidebar` component definition)

- [ ] **Step 1: Replace the Sidebar component definition**

Find the existing `Sidebar` component (~line 3149):

```js
const Sidebar = defineComponent({
  props: {
    methods:        { type: Array,  required: true },
    activeMethodId: { type: String, default: '' },
  },
  computed: {
    categories() {
      const order = ['Array','Collection','Function','Lang','Object','String'];
      const map = {};
      for (const m of this.methods) {
        if (!map[m.category]) map[m.category] = [];
        map[m.category].push(m);
      }
      return order.filter(cat => map[cat]).map(cat => ({ name: cat, methods: map[cat] }));
    },
  },
  template: `
    <nav class="sidebar">
      <template v-for="cat in categories" :key="cat.name">
        <div class="sidebar-category">{{ cat.name }}</div>
        <a
          v-for="m in cat.methods"
          :key="m.id"
          :href="'#' + m.id"
          :class="['sidebar-method', { active: m.id === activeMethodId }]"
        >{{ m.name }}</a>
      </template>
    </nav>
  `,
});
```

Replace it entirely with:

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
  template: `
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
  `,
});
```

- [ ] **Step 2: Open `index.html` in the browser and verify the feature works**

Check all of the following:
1. All 6 category sections show `−` on the right of their header (all expanded by default)
2. Clicking a category header collapses its methods — header now shows `+`
3. Clicking the same header again expands — shows `−` again
4. Multiple categories can be collapsed at the same time
5. The active method highlight (blue, bold) is unaffected whether the section is expanded or collapsed
6. Typing in the search box still works — categories with no matching methods don't appear
7. After a search clears, collapsed state is preserved

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: collapsible sidebar category sections with +/− toggle"
```
