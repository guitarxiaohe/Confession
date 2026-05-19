# Golden Rose Bouquet Transition Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the current single-rose reveal with a staged bouquet-growth sequence that uses the provided rose PNG as the 3D flower-head texture source.

**Architecture:** Keep the current Three.js scene and phase machine, but add a bouquet group that contains several stems, leaves, and rose-head groups. Build the flower heads from crossed textured planes so the PNG contributes directly to the 3D bouquet while staying lightweight enough for animation.

**Tech Stack:** Vue 3, TypeScript, Three.js, Vite asset imports

---

## File Structure

- `src/assets/mg.png`
- `src/components/ThreeScene.vue`

### Task 1: Add the bundled rose asset

**Files:**
- Create: `src/assets/mg.png`

- [ ] **Step 1: Copy the provided reference image into the Vite source tree**
- [ ] **Step 2: Confirm the file exists where ThreeScene can import it**

### Task 2: Build bouquet primitives in `ThreeScene.vue`

**Files:**
- Modify: `src/components/ThreeScene.vue`

- [ ] **Step 1: Import the rose asset URL**
- [ ] **Step 2: Add bouquet-level state and reusable builders**
- [ ] **Step 3: Build multiple stems, leaves, and bloom groups**
- [ ] **Step 4: Use the PNG to create rose-head textures for crossed planes**

### Task 3: Hook the bouquet into the reveal sequence

**Files:**
- Modify: `src/components/ThreeScene.vue`

- [ ] **Step 1: Swap the single-rose reveal for the bouquet reveal**
- [ ] **Step 2: Fade particles and cards while the bouquet grows**
- [ ] **Step 3: Stagger stem, leaf, and bloom timing for plant-like growth**
- [ ] **Step 4: Keep a subtle final idle motion on the bouquet**

### Task 4: Verify behavior

**Files:**
- Verify: `src/components/ThreeScene.vue`

- [ ] **Step 1: Run `pnpm build`**
- [ ] **Step 2: Run the local dev server and inspect the click-to-bouquet transition**
- [ ] **Step 3: Confirm the final scene shows several roses as a bouquet**
- [ ] **Step 4: Record the non-git workspace limitation instead of a commit step**
