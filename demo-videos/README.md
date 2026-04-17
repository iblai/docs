# ibl.ai Demo Videos

Product showcase videos for ibl.ai's AI platform. Three videos covering the Joseph chat assistant, MentorAI, and Agentic OS.

## Videos

| File | Product | Tool | Duration |
|------|---------|------|----------|
| `chat-demo-joseph.mp4` | Joseph — AI Campus Assistant | Remotion | ~28s |
| `mentorai-showcase.mp4` | MentorAI Showcase | HyperFrames | ~33s |
| `agentic-os-showcase.mp4` | Agentic OS Showcase | HyperFrames | ~34s |

---

## Prerequisites

- **Node.js** v22+
- **FFmpeg** — install via `winget install Gyan.FFmpeg` (Windows) or `brew install ffmpeg` (Mac)

---

## Video 1 — Chat Demo (Joseph)

Built with **[Remotion](https://www.remotion.dev/)** — React-based programmatic video. Animates Joseph responding to "What do you do?" with 12 capability bullets, scroll, typing effects, and an end card.

**Source:** `Remotion/` at the root of this project.

### Setup

```bash
cd Remotion
npm install
```

### Preview

```bash
npx remotion studio
```

Opens a browser at `localhost:3000` where you can scrub through the timeline frame by frame.

### Render

```bash
npx remotion render ChatDemo out/video.mp4
```

### Customization

All timing constants are at the top of `src/ChatDemo.tsx`:

```ts
const BULLETS_START = 255;   // frame bullets begin appearing
const BULLET_STAGGER = 28;   // frames between each bullet
const TOTAL_FRAMES = 856;    // total video length (~28.5s at 30fps)
```

To change the chat content, edit the `BULLETS` array in `src/ChatDemo.tsx`. Each entry has `emoji`, `title`, and `desc` fields.

To swap assets (logo, profile photo), replace files in `Remotion/public/`:
- `iblailogo.png` — top-left logo
- `joseph.jpg` — Joseph's avatar

---

## Videos 2 & 3 — MentorAI and Agentic OS Showcases

Built with **[HyperFrames](https://hyperframes.dev/)** — HTML+CSS+GSAP video framework. Each composition is a single `index.html` file with a GSAP timeline registered to `window.__timelines`.

**Sources:**
- `MentorAI-Showcase/index.html`
- `AgenticOS-Showcase/index.html`

### Setup

No install needed. HyperFrames runs via `npx`.

### Preview (live hot-reload)

```bash
cd MentorAI-Showcase   # or AgenticOS-Showcase
npx hyperframes preview
```

Opens the HyperFrames Studio in your browser at `localhost:3002`. The timeline scrubs in real time and hot-reloads on file save.

> **Note (Windows):** If `npx hyperframes render` says FFmpeg not found, run via node directly:
> ```bash
> node "C:/Users/<you>/AppData/Local/npm-cache/_npx/<hash>/node_modules/hyperframes/dist/cli.js" render
> ```
> Find the hash by running `npx hyperframes --version` once to cache it.

### Render

```bash
cd MentorAI-Showcase
npx hyperframes render

cd AgenticOS-Showcase
npx hyperframes render
```

Output lands in `renders/` inside each project folder.

For higher quality:

```bash
npx hyperframes render --quality high --fps 60
```

### Lint (catch issues before rendering)

```bash
npx hyperframes lint
```

Reports errors (must fix) and warnings (should fix). Common issues:
- Missing `data-composition-id` on root wrapper
- GSAP/CSS transform conflicts (use `xPercent`/`yPercent` instead of CSS `transform: translate()`)
- Scene opacity not reset after transitions

---

## Creating a New HyperFrames Video

### 1. Scaffold

```bash
npx hyperframes init my-video
cd my-video
```

### 2. Design identity

Create `DESIGN.md` before writing any HTML. The HyperFrames linter enforces this. Minimum required:

```md
## Colors
- `#060a14` — canvas
- `#4f8ef7` — brand blue
...

## Typography
- `Space Grotesk` — headlines 900

## What NOT to Do
- No gradient text
...
```

### 3. Composition structure

Every composition needs:

```html
<div data-composition-id="my-video"
     data-width="1920"
     data-height="1080"
     data-duration="30"
     data-start="0"
     style="position:relative;width:1920px;height:1080px;overflow:hidden;">

  <!-- scenes -->
  <div id="s1" class="scene">...</div>
  <div id="s2" class="scene" style="opacity:0;">...</div>

  <!-- transition wipes (optional) -->
  <div id="wipe-a" style="position:absolute;top:0;left:0;width:1920px;height:1080px;background:#4f8ef7;transform:translateX(-1920px);z-index:200;"></div>

</div>

<script>
  var tl = gsap.timeline({ paused: true });

  // ... animations ...

  // IMPORTANT: hide previous scene after each transition
  tl.set("#s1", { opacity: 0 }, transitionTime + 0.5);

  window.__timelines = window.__timelines || {};
  window.__timelines["my-video"] = tl;
</script>
```

### 4. Key rules

**Entrances:** always `gsap.from()` — animate FROM offscreen TO the CSS end-state position. Never animate TO a position you guessed.

**Exits:** only the final scene should fade out. All other scenes exit via their transition effect.

**Scene cleanup:** after every transition, set the outgoing scene to `opacity: 0`. If you forget, its content bleeds through subsequent scenes.

```js
// After a wipe transition reveals s2, hide s1:
tl.set("#s1", { opacity: 0 }, T12 + 0.5);
```

**GSAP/CSS transform conflicts:** never put `transform: translate(-50%, -50%)` on an element that GSAP will also animate. Use `gsap.set()` for initial centering instead:

```js
// Instead of CSS: transform: translate(-50%, -50%) scale(0)
gsap.set("#my-element", { xPercent: -50, yPercent: -50, scale: 0 });
```

**Easing:** entrances use `expo.out`, `power3.out`, or `power4.out`. Never `ease-in` for entrances.

**Infinite loops:** HyperFrames doesn't support `repeat: -1`. Use finite repeat counts (`repeat: 3`) or compute how many iterations fit in the scene duration.

### 5. Embedding local images

HyperFrames' file server only serves files relative to the project directory. To embed a PNG logo reliably, convert it to a base64 data URI:

```js
// In Node.js — run once, paste output into your HTML
const b64 = require('fs').readFileSync('logo.png').toString('base64');
console.log('data:image/png;base64,' + b64);
```

Then use it as the `src` of your `<img>` tag directly in the HTML.

---

## Design System

All three videos use the ibl.ai brand palette:

| Token | Hex | Role |
|-------|-----|------|
| Canvas | `#060a14` | Background |
| Text | `#f0f4ff` | Primary copy |
| Blue | `#4f8ef7` | Brand accent |
| Cyan | `#06b6d4` | Energy accent (1 element/scene max) |
| Gold | `#fbbf24` | Stats and proof numbers only |
| Surface | `#0d1a2e` | Cards and panels |
| Muted | `#64748b` | Secondary copy |

**Fonts:** `Space Grotesk` (headlines, weight 700–900) + `Space Mono` (labels, data, monospaced elements). Both loaded automatically by the HyperFrames compiler.
