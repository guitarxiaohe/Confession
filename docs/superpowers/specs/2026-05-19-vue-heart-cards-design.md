# Vue Heart Cards Design

**Date:** 2026-05-19

**Goal:** Build a standalone `Vue 3 + TypeScript + Vite` page where many pastel text cards animate in from the edges of the screen to form a glowing heart shape. Hovering a card should lift it out of the heart, move it to the center of the screen, enlarge it for reading, and return it to the heart when the pointer leaves.

## Scope

- Build a single-page experience focused on one visual interaction.
- Use text-only pastel cards instead of image cards.
- Use real DOM cards so hover, text rendering, and future extension stay simple.
- Do not add routing, backend logic, or unrelated controls.

## Visual Direction

- Dark romantic backdrop with a soft neon glow around the heart.
- Dozens of pastel rounded cards with short positive Chinese phrases.
- Cards form a layered heart outline rather than a flat solid fill.
- The finished composition should feel light, dreamy, and clean.

## Technical Direction

- Use `Vue 3 + TypeScript + Vite`.
- Use absolutely positioned DOM cards inside a full-screen stage.
- Compute a set of target points on a heart curve and assign one card per point.
- Animate cards from random positions around the viewport into those heart points.
- On hover, temporarily override one card's transform so it flies to the center and scales up.

## Interaction Model

### Initial Load

- Every card starts off-screen or near a screen edge.
- Cards animate inward with staggered timing, slight rotation, and easing.
- When the sequence finishes, the cards settle into a stable heart.

### Hover

- Hovering a settled card raises its z-index above the rest.
- The hovered card leaves the heart and flies to the center of the screen.
- The centered card scales up for easier reading.
- Non-hovered cards remain in the heart so the overall composition stays visible.

### Hover Exit

- The enlarged card returns smoothly to its original heart position.
- The heart returns to its full stable arrangement without re-running the load animation.

## Layout Model

- The stage fills the viewport.
- Heart points are calculated from a parametric heart curve.
- The heart scales responsively with viewport size.
- Cards receive small per-card rotation offsets and depth-like stacking to mimic the layered reference effect.

## File Plan

- `package.json`: Vite project metadata and scripts.
- `tsconfig*.json`: TypeScript config for Vite.
- `vite.config.ts`: Vite config.
- `index.html`: Vite entry HTML.
- `src/main.ts`: App bootstrap.
- `src/App.vue`: Full-screen stage and interaction orchestration.
- `src/components/HeartCard.vue`: Card presentation component.
- `src/composables/useHeartLayout.ts`: Heart point generation and transform helpers.
- `src/data/messages.ts`: Pastel card message data.
- `src/style.css`: Global page styling and ambient backdrop.

## Validation

- Dev server starts successfully.
- Build succeeds successfully.
- On first load, cards visibly fly in from screen edges and settle into a heart.
- Hovering a card moves it to the center and enlarges it.
- Moving the pointer away returns the card to the heart.
- Layout remains centered and usable on common desktop and laptop viewport sizes.

## Notes

- This workspace is not a Git repository, so the design document cannot be committed here.
