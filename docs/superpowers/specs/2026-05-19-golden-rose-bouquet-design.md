# Golden Rose Bouquet Transition Design

**Date:** 2026-05-19

**Goal:** Extend the existing Three.js heart-card scene so that clicking the `点我` button fades the cards and particles away, then grows a golden rose bouquet from the bottom of the scene like a living plant. The bouquet should feel layered and luminous, with multiple blossoms rather than a single rose.

## Scope

- Keep the current scene entry flow: scattered particles, text, heart, and card arrangement.
- Preserve the existing `点我` reveal trigger.
- Replace the single-rose reveal with a multi-rose bouquet reveal.
- Use the provided `mg.png` as the visual reference and flower-head texture source.
- Do not add new routes or unrelated UI.

## Visual Direction

- Mood: dreamlike, precious, ceremonial.
- Palette: warm gold, champagne, amber highlights, and soft ivory bloom.
- Structure: one main central bloom plus several secondary blooms arranged as a bouquet.
- Motion: stems grow upward first, leaves open next, then blossoms expand one after another.

## Technical Direction

- Reuse the existing Three.js scene and animation state machine in `ThreeScene.vue`.
- Add a bouquet group composed of:
  - multiple rose stems and leaves built from lightweight 3D geometry
  - multiple rose-head groups arranged in 3D bouquet space
  - rose heads built from crossed textured planes derived from the provided PNG so the reference image is used in the 3D scene
- Keep cards and particles as the pre-reveal layer, then fade them while the bouquet grows in.

## Reveal Timeline

1. **Fade phase**
   - Cards, glows, and particle systems lose opacity gradually.
   - The heart recedes into the background instead of disappearing abruptly.
2. **Stem phase**
   - A main stem grows up from the lower center.
   - Side stems branch outward with slight angle variation.
3. **Leaf phase**
   - Leaves unfold from side stems with a delayed scale-and-rotate motion.
4. **Bloom phase**
   - The center bloom opens first.
   - Secondary blooms appear in staggered order around it.
5. **Hold phase**
   - The completed bouquet breathes subtly and glints under soft glow.

## File Plan

- `src/assets/mg.png`: local copy of the reference image for bundling.
- `src/components/ThreeScene.vue`: bouquet asset loading, bouquet construction, animation timing, and reveal sequencing.

## Validation

- The app still builds successfully.
- Clicking `点我` starts the bouquet reveal.
- Cards and particles fade away during the transition.
- Several roses appear, not just one.
- The bouquet grows progressively from bottom to top.
- The final bouquet remains centered and visually layered.

## Notes

- The provided PNG is used as a rose-head texture source and appearance reference, not converted into true sculpted polygonal petals.
- This workspace is not a Git repository, so the design document cannot be committed here.
