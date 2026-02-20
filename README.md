# Marketing AI Framework

> **Live site:** [fiskerm.github.io/Marketing_AI_Map](https://fiskerm.github.io/Marketing_AI_Map/)

An interactive web app that maps **40 AI marketing use cases** across **8 themes** — from quick wins to fully AI-embedded operations. Built as a consultancy tool to help clients understand where they are on their AI maturity journey and what to prioritise next.

---

## Table of Contents

- [What This Project Does](#what-this-project-does)
- [Project Structure](#project-structure)
- [How the Files Work Together](#how-the-files-work-together)
- [File 1: index.html — The Unified Shell](#file-1-indexhtml--the-unified-shell)
- [File 2: dependency-map.html — The Detail View](#file-2-dependency-maphtml--the-detail-view)
- [Key Concepts for Junior Devs](#key-concepts-for-junior-devs)
- [How to Make Changes](#how-to-make-changes)
- [Deployment](#deployment)

---

## What This Project Does

This project has two main visualisations:

1. **The Octagon** — A radial diagram showing all 8 marketing themes arranged in a circle. Click any segment to see what it depends on and what it unlocks. Think of it as the "big picture" view.

2. **The Dependency Map** — A detailed card-based view where you can drill into each of the 40 individual use cases, see their difficulty levels, and explore the exact dependency chains between them.

Both views share the same underlying data: 8 themes, each with 5 use cases (except Measurement which has 6), totalling 40+ use cases.

---

## Project Structure

```
├── index.html               ← The homepage (shell with nav + Octagon)
├── dependency-map.html       ← The detailed dependency map (loaded via iframe)
├── marketing-ai-build-guide.docx  ← Original specification document
└── README.md                 ← You are here
```

**There are no external CSS or JS files.** Everything is self-contained inside each HTML file. The only external dependency is **Google Fonts** (loaded via CDN).

---

## How the Files Work Together

```
┌─────────────────────────────────────────────────────────┐
│  index.html (the shell)                                 │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Top Nav Bar                                      │  │
│  │  [● Octagon Overview]  [● Dependency Map]         │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  View 1 (default): Octagon rendered inline              │
│  View 2 (on click): dependency-map.html in <iframe>     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Why an iframe instead of merging everything into one file?**

Both pages have their own `<style>` and `<script>` blocks with CSS class names and JavaScript variables that would collide if merged. For example, both files style `body` differently, and both define data arrays called `THEMES` / `themes`. The iframe keeps them cleanly isolated — no CSS leaks, no variable conflicts. This is a common pattern when combining two standalone apps into one interface.

---

## File 1: `index.html` — The Unified Shell

This is the homepage and entry point. It has three main parts:

### Part 1: The Navigation Bar (HTML, lines ~245–265)

```html
<nav class="top-nav">
  <div class="nav-brand">Marketing AI Framework</div>
  <div class="nav-tabs">
    <button class="nav-tab active" onclick="switchView('octagon')">Octagon Overview</button>
    <button class="nav-tab" onclick="switchView('map')">Dependency Map</button>
  </div>
</nav>
```

- Fixed to the top of the screen (`position: fixed`)
- Uses `backdrop-filter: blur(16px)` to create a frosted glass effect
- The `active` class controls which tab is highlighted
- Clicking a tab calls `switchView()` which toggles between the two views

### Part 2: The Octagon Visualisation (SVG + Canvas)

This is the most technically interesting part. The octagon is built using **two overlapping layers**:

#### Layer 1: SVG — The coloured segments

```
SVG viewBox: -380 -380 760 760  (centred at 0,0)
```

Each of the 8 theme segments is drawn as an SVG `<path>` using **arc mathematics**. Here is what the code does:

1. Divides a full circle (`2π radians`) into 8 equal sectors
2. For each sector, calculates the start angle (`s`) and end angle (`e`)
3. Draws a wedge shape from an inner radius (`R_IN = 74px`) to an outer radius (`R_OUT = 372px`)
4. The `GAP = 0.024 radians` creates thin spacing between segments

The SVG path command looks like this conceptually:

```
M (inner start) → Arc along inner circle → L (outer end) → Arc along outer circle → Z (close)
```

#### Layer 2: Canvas — The text labels

Why not put text in the SVG? Because **text in a radial layout needs to be rotated per-segment**, and some segments are on the bottom half of the circle where text would appear upside-down. The canvas gives us pixel-level control over rotation.

Key logic in `drawAll()`:

```javascript
// Segments on the bottom half (indices 3,4,5,6) need flipping
const FLIP_INDICES = [3, 4, 5, 6];
const isLeftSide = FLIP_INDICES.includes(i);

// If on the left/bottom, rotate text an extra 180° so it reads left-to-right
let pillRot = mid + Math.PI / 2;
if (isLeftSide) pillRot += Math.PI;
```

Each use case is drawn as a small rounded rectangle ("pill") with:
- A **coloured difficulty dot** (green = easy, yellow = medium, red = hard)
- The **use case name** (truncated with `…` if it doesn't fit)

#### The Data Model

```javascript
const THEMES = [
  {
    id: 'creative',                    // Unique identifier
    name: 'Creative',                  // Display name
    color: '#00F5C4',                  // Theme colour
    useCases: ['Copy from brief', ...], // 5 use cases (titles only)
    needs: ['audience', 'insights'],   // Theme-level dependencies
    enables: ['content', 'media']      // What this theme unlocks
  },
  // ... 7 more themes
];
```

**Important:** In this file, dependencies are at the **theme level** (Creative needs Audience), not at the individual use case level. The dependency map has use-case-level dependencies.

### Part 3: Interaction Logic

**`handleClick(id)`** — When you click a segment:
1. Sets `activeId` to the clicked theme
2. Calculates which other themes are related (`needs` + `enables`)
3. Everything else gets added to `dimmedIds` — these are visually faded out
4. Updates SVG fill/stroke to highlight the active segment and its connections
5. Shows a **dependency panel** on the right listing "Needs first ▲" and "Unlocks ▼"
6. Redraws the canvas with `drawAll()` (dimmed segments get low opacity)

**`resetAll()`** — Clears the selection and restores all segments to their default state.

**`switchView(view)`** — Toggles between the Octagon and the Dependency Map:
```javascript
function switchView(view) {
  // Remove 'active' from all tabs and views
  // Add 'active' to the selected tab and view
  
  // Lazy-load the iframe on first click (performance optimisation)
  if (view === 'map' && !mapLoaded) {
    const iframe = document.createElement('iframe');
    iframe.src = 'dependency-map.html';
    document.getElementById('map-view').appendChild(iframe);
    mapLoaded = true;
  }
}
```

### Part 4: Responsive Scaling

The octagon is designed at a fixed `1440×810px` (standard presentation dimensions). To make it work on any screen size:

```javascript
function scaleSlide() {
  const availH = window.innerHeight - 52; // subtract nav bar height
  const scale = Math.min(window.innerWidth / 1440, availH / 810);
  document.getElementById('octagon-slide').style.setProperty('--slide-scale', scale);
}
```

This uses a CSS custom property (`--slide-scale`) with `transform: scale()` to shrink the entire slide proportionally. The `Math.min()` ensures it fits both width and height without cropping.

---

## File 2: `dependency-map.html` — The Detail View

This is the detailed, card-based view of all 40 use cases. It has a completely different design language (using `Syne` and `Space Mono` fonts instead of `Inter`).

### Layout

```
┌──────────┬───────────────────────────────────────────┐
│ Sidebar  │  Content Area                             │
│          │                                           │
│ Overview │  Shows cards for the selected theme       │
│ Creative │  Each card = one use case                 │
│ Audience │                                           │
│ Content  │  Click a card to expand it and see:       │
│ Media    │  - Description                            │
│ Insights │  - What it needs (▲)                      │
│ Distrib. │  - What it enables (▼)                    │
│ Measure. │                                           │
│ Journey  │                                           │
└──────────┴───────────────────────────────────────────┘
```

### Data Model (more detailed than the Octagon)

```javascript
const themes = {
  creative: {
    name: "Creative",
    num: "01",
    useCases: [
      {
        id: "cr1",
        level: 1,                                    // Difficulty: 1-5
        title: "AI copy generation from brief",
        desc: "Standalone — just needs a brief...",   // Human explanation
        needs: [],                                    // Use case IDs this depends on
        enables: ["cr2", "cr4", "co2", "di2"]        // Use case IDs this unlocks
      },
      // ... more use cases
    ]
  }
};
```

**Key difference from `index.html`:** Dependencies here are at the **individual use case level** (e.g., `cr2` needs `cr1`), not the theme level.

### ID Convention

Each use case has a 2-letter + number ID:

| Prefix | Theme | Examples |
|--------|-------|----------|
| `cr` | Creative | `cr1`, `cr2`, `cr3`, `cr4`, `cr5` |
| `au` | Audience | `au1`–`au5` |
| `co` | Content | `co1`–`co5` |
| `me` | Media & Performance | `me1`–`me5` |
| `in` | Insights & Intelligence | `in1`–`in5` |
| `di` | Distribution & Channel | `di1`–`di5` |
| `ma` | Measurement & Attribution | `ma0`–`ma5` |
| `jo` | Customer Journey | `jo1`–`jo5` |

### Key Functions

**`renderCard(uc)`** — Generates the HTML for one use case card. Uses template literals to build the card dynamically:
- Shows a difficulty badge (colour-coded by level)
- Lists dependency tags you can click to navigate
- Has an expandable info panel

**`toggleCard(id)`** — Expands/collapses a card and highlights its dependency chain:
- The clicked card gets class `highlighted` (green border)
- Cards it depends on get class `dependency` (yellow border)
- Cards it enables get class `dependent` (red border)

**`jumpTo(id)`** — Cross-theme navigation. If you click a dependency tag that belongs to a different theme:
1. Switches the sidebar to that theme
2. Waits 50ms for the DOM to update
3. Expands the target card
4. Smooth-scrolls to it

**`showTheme(key)`** — Renders all cards for a given theme, or the overview page with summary cards and a "Critical Path" diagram.

### The Overview Page

When "Overview" is selected, the content area shows:
- A **Critical Path** — the recommended sequence of use cases from quick wins to full AI maturity
- **8 summary cards** (one per theme) showing average dependencies and difficulty

---

## Key Concepts for Junior Devs

### CSS Custom Properties (Variables)

`dependency-map.html` uses `:root` variables for consistent theming:

```css
:root {
  --bg: #0a0a0f;
  --accent1: #00f5c4;   /* green — used for "easy" and "enables" */
  --accent2: #ff4d6d;   /* red — used for "hard" and "dependent" */
  --accent3: #7b61ff;   /* purple — used for dependency links */
  --accent4: #ffd166;   /* yellow — used for "medium" and "needs" */
}
```

This means you can change the entire colour scheme by editing these 5 lines.

### SVG Path Commands

The Octagon uses SVG path `d` attributes with these commands:
- `M x y` — Move to a point (pen up)
- `A rx ry rotation large-arc sweep x y` — Draw an arc
- `L x y` — Draw a line to a point
- `Z` — Close the path

### Canvas 2D API

The Octagon's text is rendered using the HTML Canvas API:
- `ctx.save()` / `ctx.restore()` — Save and restore the drawing state (position, rotation, opacity)
- `ctx.translate(x, y)` — Move the origin to a new position
- `ctx.rotate(angle)` — Rotate the coordinate system
- `ctx.fillText(text, x, y)` — Draw text
- `ctx.globalAlpha` — Control transparency (0 = invisible, 1 = fully visible)

### Template Literals

Both files use backtick strings (`` ` ``) to build HTML dynamically:

```javascript
html += `<div class="dep-item" onclick="handleClick('${id}')">
  <div class="dep-dot" style="background:${theme.color}"></div>
  <div class="dep-text">${theme.name}</div>
</div>`;
```

The `${...}` parts are JavaScript expressions that get inserted into the string. This is how the app turns data into visible HTML elements.

### Lazy Loading

The dependency map iframe is only created when the user first clicks the tab:

```javascript
if (!mapLoaded) {
  const iframe = document.createElement('iframe');
  iframe.src = 'dependency-map.html';
  mapLoaded = true;
}
```

This is a performance pattern — don't load resources until they're needed.

---

## How to Make Changes

### Adding or editing a use case

1. Open `dependency-map.html`
2. Find the theme in the `themes` object (line ~411)
3. Add/edit the use case object in the `useCases` array
4. Make sure the `id` follows the naming convention (`cr6`, `au6`, etc.)
5. Update any `needs`/`enables` arrays in other use cases that reference it

If you also want it to appear in the Octagon (`index.html`), update the `useCases` array in the matching `THEMES` entry — but note that the Octagon only shows 5 use cases per theme.

### Changing colours

- **Octagon:** Edit the `color` property in the `THEMES` array, or the `D_COLORS` array for difficulty dots
- **Dependency Map:** Edit the CSS variables in `:root` at the top of `dependency-map.html`

### Adding a new theme

This is a larger change:
1. Add the theme data to both files
2. In `index.html`, change `const N = 8` to `9` (the octagon becomes a nonagon)
3. Update the `FLIP_INDICES` array — you'll need to figure out which segments are on the bottom half
4. Add a new sidebar button in `dependency-map.html`

---

## Deployment

This project is hosted on **GitHub Pages** — a free static site hosting service from GitHub.

### How it works

1. Code is pushed to the `main` branch of [github.com/FiskerM/Marketing_AI_Map](https://github.com/FiskerM/Marketing_AI_Map)
2. GitHub Pages automatically serves the files as a website
3. `index.html` is the default page (GitHub Pages looks for this filename)
4. The site updates within ~60 seconds of a push

### To update the live site

```bash
# Stage your changes
git add .

# Commit with a descriptive message
git commit -m "Description of what changed"

# Push to GitHub (triggers automatic deployment)
git push
```

### Live URL

```
https://fiskerm.github.io/Marketing_AI_Map/
```
