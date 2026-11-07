# Resume banner demo

Vue 3 + Tailwind CSS 4 + Vite. Resume text and artwork are illustrative. Hosted on Netlify; source maintained on GitHub.

## Run

```sh
npm install
npm run dev
```

## Controls

Move the mouse alone: no animation. Click/tap to nudge the badge. Grab anywhere on the card and drag left, right, up, or down; the grab offset is preserved. Release to let gravity and an elastic tether pull it back into a damped swing. Holding still before release removes throw momentum. Touch and Enter/Space activation are supported. Reduced-motion preferences keep direct dragging but reset on release.

## Implementation

Edit src/App.vue for content and motion. The card attachment uses two-dimensional position and velocity, gravity, and a tension-only spring. A cubic SVG ribbon follows the fixed ceiling anchor and moving clip, with slack curvature when lifted. The card and hit surface use 2D transforms. This is a lightweight visual approximation, not full cloth simulation.

## Build

```sh
npm run build
npm run preview
```

Browser checks cover four drag directions and return motion, touch input, nine grab positions, pointer-follow remaining disabled, click/keyboard activation, reduced motion, and mobile overflow.

Motion tuning: gravity 400 and tether stiffness 50 preserve the resting length while slowing recoil. Radial damping 6 absorbs strap stretch; swing damping 0.65 lets lateral motion decay over several passes. Card rotation uses stiffness 32 and damping 10 for a softer follow-through.

## Continuous deployment

Pushes to GitHub branch `main` automatically trigger a Netlify production build (`npm run build`, output `dist`, Node.js 22). Netlify reads the private repository using a read-only deploy key; a GitHub push webhook starts the build.
