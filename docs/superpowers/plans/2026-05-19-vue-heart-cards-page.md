# Vue Heart Cards Page Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a `Vue 3 + TypeScript + Vite` single page where pastel text cards fly in from screen edges to form a heart and a hovered card flies to the center for enlarged reading.

**Architecture:** Use a small Vite app with a full-screen stage in `App.vue`, a focused `HeartCard` presentational component, message data in a separate module, and a composable that computes responsive heart positions and initial edge positions. Drive motion with CSS transitions and per-card transform strings rather than canvas or 3D libraries.

**Tech Stack:** Vue 3, TypeScript, Vite, CSS

---

## File Structure

- `package.json`
- `tsconfig.json`
- `tsconfig.app.json`
- `vite.config.ts`
- `index.html`
- `src/main.ts`
- `src/App.vue`
- `src/components/HeartCard.vue`
- `src/composables/useHeartLayout.ts`
- `src/data/messages.ts`
- `src/style.css`

### Task 1: Scaffold the minimal Vite app

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `tsconfig.app.json`
- Create: `vite.config.ts`
- Create: `index.html`
- Create: `src/main.ts`

- [ ] **Step 1: Create the Vite project files**
- [ ] **Step 2: Install dependencies with `pnpm install`**
- [ ] **Step 3: Confirm the dependency install succeeds**

### Task 2: Build the card data and layout helpers

**Files:**
- Create: `src/data/messages.ts`
- Create: `src/composables/useHeartLayout.ts`

- [ ] **Step 1: Add a message list large enough to draw a heart**
- [ ] **Step 2: Implement responsive heart point generation**
- [ ] **Step 3: Implement random edge start positions and transform helpers**

### Task 3: Implement the stage and card components

**Files:**
- Create: `src/components/HeartCard.vue`
- Create: `src/App.vue`
- Create: `src/style.css`

- [ ] **Step 1: Build the reusable card component**
- [ ] **Step 2: Build the full-screen stage and map cards to transforms**
- [ ] **Step 3: Implement first-load staggered fly-in behavior**
- [ ] **Step 4: Implement hover-to-center and return behavior**
- [ ] **Step 5: Add backdrop glow and layered visual styling**

### Task 4: Verify the app

**Files:**
- Verify: `src/App.vue`
- Verify: `src/components/HeartCard.vue`
- Verify: `src/composables/useHeartLayout.ts`

- [ ] **Step 1: Run `pnpm build`**
- [ ] **Step 2: Run the dev server briefly and confirm it starts**
- [ ] **Step 3: Check the key requirements against the approved design**
- [ ] **Step 4: Record the non-git workspace limitation instead of a commit step**
