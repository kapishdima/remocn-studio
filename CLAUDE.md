# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

@AGENTS.md

## Version warning (read this first)

This project pins **Next.js 16.2.9** and **React 19.2.4** — both newer than most training data. APIs, file conventions, and async behavior differ from older Next.js. Per `AGENTS.md`, **read the relevant guide under `node_modules/next/dist/docs/` before writing any Next.js code** (`01-app`, `02-pages`, `03-architecture`). Heed deprecation notices there.

## Commands

The lockfile is `bun.lock`; use bun.

- `bun dev` — start dev server (http://localhost:3000)
- `bun run build` — production build
- `bun start` — serve the production build

There is no test runner, linter, or formatter configured. Type checking happens through the build (`tsconfig.json` is `noEmit`); run `bun run build` to surface type errors.

To add UI components: `bunx shadcn@latest add <component>` (config in `components.json`).

## Architecture

This is a shadcn/ui component library scaffold on the Next.js App Router.

- **UI primitives are `@base-ui/react`, NOT Radix.** Every component in `components/ui/` imports from `@base-ui/react/*` (e.g. `import { Button as ButtonPrimitive } from "@base-ui/react/button"`). Base UI's component APIs, prop names, and composition patterns differ from Radix — do not assume Radix props/slots. Check the actual primitive import before editing a component.
- **Styling is Tailwind v4, CSS-first.** There is no `tailwind.config.*`. All theme config lives in `app/globals.css` via `@import "tailwindcss"`, `@theme inline { ... }`, and CSS variables (`:root` / `.dark`). Colors use `oklch()`. The radius scale is derived (`--radius-sm` … `--radius-4xl` computed from `--radius`). `shadcn/tailwind.css` is imported there too.
- **shadcn config** (`components.json`): style `base-luma`, base color `zinc`, RSC enabled, Lucide icons. Variants are defined with `class-variance-authority` (`cva`) inside each component.
- **`cn()`** in `lib/utils.ts` (clsx + tailwind-merge) is the standard className composer — use it for all conditional/merged classes.
- **Path alias** `@/*` maps to the repo root. Aliases: `@/components`, `@/components/ui`, `@/lib`, `@/lib/utils`, `@/hooks`.
- **Fonts** are loaded via `next/font/google` in `app/layout.tsx` (Geist, Geist Mono, Manrope) and exposed as CSS variables (`--font-geist-sans`, `--font-geist-mono`, `--font-heading`) wired into the `@theme` block.

## Layout / structure

- `app/` — App Router entry (`layout.tsx`, `page.tsx`, `globals.css`).
- `components/ui/` — the generated shadcn component set (Base UI–backed).
- `lib/utils.ts` — `cn` helper.
- `hooks/` — shared hooks (e.g. `use-mobile.ts`).


# remocn studio — design

A standalone, hosted in-browser **video editor** that assembles videos from first-party
remocn components: drag components onto a typed timeline, configure them, preview in the
browser, and render an MP4 server-side.

> Status: design. This document is the source of truth for the build.
> The studio is a **separate project** (`remocn-studio`) deployed at **studio.remocn.dev**,
> not part of this repo. remocn.dev itself is the landing page.

---

## 1. What it is

- **Hosted** on `studio.remocn.dev`, against the **first-party remocn component set only**
  (a closed, known catalog — not the user's local components).
- Users assemble a video from remocn components, preview via `@remotion/player`, and render
  an MP4 on the server.
- **v1 ships MP4 download.** "Own your code" (export readable Remotion `.tsx`) is a designed-for
  v2 feature — the JSON project format keeps code-gen possible.
- **remocn.dev is the landing page** and owns the "Open Studio" CTA. The studio repo has **no
  marketing pages** — only the editor, dashboard, and auth.

## 2. Timeline model — typed 3-track

The component taxonomy dictates the timeline shape. Components are not uniform clips:

| Track | Holds | Render mapping |
|-------|-------|----------------|
| **BG** | one background spanning the whole video (`mesh-gradient-bg`, `volumetric-rays`) | `<AbsoluteFill>` behind everything |
| **Scene** | full scenes/compositions back-to-back (`claude-chat`, `terminal-simulator`), with **transitions docked on the seams** | sequential clips + custom cross-fade engine over seam overlaps |
| **Overlay** | floating, timed elements (`typewriter`, badges), **freely placed on the canvas** | `<Sequence from/durationInFrames>` + `{x,y,w,h}` |

```
BG     │███████████████████████│ mesh-gradient
SCENE  │█claude-chat█▒fade▒█terminal█│
OVERLAY│   █typewriter█      █badge█ │
```

## 3. Project format — the spine

A project is **one JSON document**. It feeds two consumers that must stay identical:

```
project.json
{ fps, width, height,
  tracks: [
    { kind: 'bg',    clips: [{ componentId, props }] },
    { kind: 'scene', clips: [
        { componentId, from, durationInFrames, props },
        { transition: { type, overlapFrames } },
        { componentId, durationInFrames, props } ] },
    { kind: 'overlay', clips: [
        { componentId, from, durationInFrames, props, x, y, w, h } ] } ] }
        │
        ├─▶  <Player component={ProjectRoot} inputProps={project} />     (client preview)
        └─▶  renderMedia(serveUrl, "studio-project", inputProps=project) (server render)
```

- **`ProjectRoot`** is one generic Remotion component that reads the JSON, looks each
  `componentId` up in `component-map.generated.ts`, and renders BG / Scene (with the transition
  engine) / Overlay.
- `ProjectRoot` + the component map are **shared code** used by both the client Player and the
  render bundle — that is what makes preview and render provably identical.
- `calculateMetadata` derives total duration / fps / dimensions from the JSON.

## 4. Component manifest — extend `config.ts`, do NOT add zod

Every registry component already ships a `config.ts` typed as `ComponentConfig`
(`lib/customizer-config.ts`) that powers the docs customizer. It already encodes what the studio
needs — reuse it instead of retrofitting zod schemas:

```ts
// existing fields, already authored per component:
controls: { fromText: { type:'text', default:'…', label:'…' }, … }  // right-panel form schema
durationInFrames, fps, compositionWidth, compositionHeight, previewBackdrop, componentName, importPath

// add for studio:
track: 'bg' | 'scene' | 'overlay'
durationMode: 'fixed' | 'stretchable'
category: string
studio?: boolean   // opt-in; the palette = installed components with studio === true
```

- The studio's **right-panel inspector renders from `config.controls`** — reuse the docs
  customizer's control→widget renderer (`lib/ui-preview-internals.tsx`) verbatim.
- `durationInFrames` is the clip's **default duration**.
- **zod is used only as an optional server-side validation guard** on the incoming project JSON,
  never to generate the UI.

## 5. Transitions & duration

- **Custom cross-fade engine in `ProjectRoot`.** Transitions are seam **data**
  (`{ type, overlapFrames }`); `ProjectRoot` computes per-frame opacity/transform across the
  overlap window for the two adjacent scenes.
- The registry's existing "transition" components (`fade-through`, wipes) are **self-contained
  scene clips** (they morph their own content A→B with hardcoded frame timings) — they are **not**
  `@remotion/transitions` `TransitionPresentation`s and stay on the **Scene track as standalone
  clips**. `@remotion/transitions` is not used.
- **Most clips are fixed-duration** (hardcoded timings; stretching longer holds the clamped final
  state, shortening truncates — acceptable for v1). `stretchable` is opt-in for components that
  derive timing from `useVideoConfig().durationInFrames`.

## 6. Overlay placement — drag-on-canvas

- Overlays are placed by **direct manipulation on the Player**: selection handles, drag-to-move,
  corner-scale, snap guides. Schema stores `{ x, y, w, h }`.
- Screen→composition coords via `PlayerRef.getScale()`; the handle layer sits above the Player;
  **pause playback during drag**. (Heaviest single UI item in the build.)

## 7. Content scope (v1)

- First-party components **+ user text only** (edit text props: messages, headlines, code).
- **No** user image/video/audio uploads, **no** audio track. (→ no object storage, renders stay
  stateless and fast; the bundle already contains every component.)
- Uploads + audio track are a clearly-scoped v2.

## 8. Topology — standalone repo consuming the registry

```
remocn (this repo)                         remocn-studio (new repo)
 registry/remocn/<c>/                         components/remocn/<c>.tsx        ← shadcn add @remocn/<c>
   index.tsx     ── publishes ──▶             components/remocn/<c>.config.ts
   config.ts  (ADD to files[])                lib/remocn-ui/   (shared customizer-config)
                                              lib/studio/component-map.generated.ts ← codegen scans configs
                                              src/remotion/{ProjectRoot,Root,index}
                                              app/(studio)/   editor UI
```

The studio is the **biggest consumer of the remocn registry** — it installs the first-party set
via shadcn (dogfooding), rather than reaching into this repo's internals.

**Prerequisite change in THIS repo:** add `config.ts` to each studio-eligible item's registry
`files[]` (today only `index.tsx` ships), plus a shared `remocn-ui` registry item for
`customizer-config.ts` (so `ComponentConfig`, `FPS`, `W`, `H`, `FONT_WEIGHT_OPTIONS` resolve after
install). Then a studio component travels complete.

`scripts/sync-registry.mts` runs `shadcn add @remocn/<set>` then codegens
`component-map.generated.ts` from installed configs with `studio: true`. Re-running it is the
**registry sync/versioning seam** (pin a version or re-sync deliberately).

## 9. Tech stack

- **Next.js (App Router)** + **shadcn/ui** + **Tailwind v4**, themed with remocn's tokens
  (copy `app/globals.css` design tokens for visual consistency).
- **@remotion/player** + **@remotion/bundler** + **@remotion/renderer**, all pinned **4.0.473**
  to match the registry components; `acknowledgeRemotionLicense` set.
- **jotai** (editor state) + **react-resizable-panels** (editor layout).
- **better-auth + Drizzle + Postgres** (self-hosted on Coolify).

## 10. Repo layout

```
remocn-studio/
  app/
    page.tsx                  → redirect: authed ? /studio/projects : /login
    login/                    minimal better-auth sign-in
    (studio)/
      studio/                 the editor (palette · timeline · canvas · inspector)
      studio/projects/        "My projects" dashboard (default authed landing)
      studio/p/[id]/          share view (Player + "made with remocn"; public if visibility=public)
    api/
      render/  ([jobId]/, download/)   generalized job queue
      auth/[...all]/                   better-auth handler
      projects/                        CRUD
  components/
    remocn/                   shadcn-installed first-party components + .config.ts
    ui/                       shadcn/ui primitives
    studio/                   timeline, clip, palette, inspector, canvas-handles
  lib/
    remocn-ui/                shared customizer-config (installed from registry)
    studio/
      project-schema.ts       project JSON shape (zod, server-validated)
      component-map.generated.ts
      transition-engine.ts
    server/                   bundle.ts, render.ts, render-queue.ts (lifted + generalized)
    db/                       drizzle schema + client
    auth.ts                   better-auth (Drizzle adapter)
  src/remotion/
    ProjectRoot.tsx · Root.tsx (registers studio-project) · index.ts
  scripts/
    sync-registry.mts · bundle-remotion.mts · ensure-browser.mts
  Dockerfile · drizzle.config.ts · components.json (registries: @remocn)
```

## 11. Backend

- **Drizzle schema:** better-auth tables (`users`/`sessions`/`accounts`) +
  `projects { id, userId, title, json (jsonb), thumbnailUrl?, visibility, createdAt, updatedAt }`.
- **better-auth** with the Drizzle adapter; GitHub/Google OAuth.
- **Render API** (lifted from this repo's `lib/server/*` + `api/render/*`): authed
  `POST /api/render` (project JSON) → `enqueueRender` → `202 { jobId }`; poll
  `GET /api/render/[jobId]`; `download`. **Server-side validation before render:** every
  `clip.componentId ∈ map` and props validated against that component's config/zod — reject
  otherwise (injection/abuse guard). Soft per-user daily render caps on the queue.

## 12. Deployment (Coolify, subdomain)

- Two Coolify services in one project: the Next.js app (Docker image with the Remotion pre-bundle
  baked in, as in this repo) and a **Postgres** container.
- Envs: `DATABASE_URL`, better-auth secrets, OAuth creds, `REMOTION_CONCURRENCY`,
  `RENDER_MAX_CONCURRENT`, `RENDER_TIMEOUT_MS`.
- Point `studio.remocn.dev` at the app service (Coolify domain → Traefik). Render box sizing
  carries over from the existing CPX42 tuning.
- Cross-domain: two independent apps, no shared session needed for v1. Optional future SSO via a
  shared `.remocn.dev` cookie domain (better-auth supports it).

## 13. Reuse vs build

**Lifted from this repo (copy + generalize):** `lib/server/{bundle,render,render-queue,cleanup,
rate-limit,paths}.ts`, `app/api/render/*`, `scripts/{bundle-remotion,ensure-browser}.mts`, the
docs customizer control renderer + Player wiring, the autoplay shim, the theme tokens.

**New:** the editor (timeline drag/resize/snap via pointer events, palette, inspector, drag-on-
canvas overlay handles), `ProjectRoot` + transition engine, `project-schema.ts`, the
`sync-registry` codegen, better-auth + Drizzle + projects CRUD + dashboard. The `ComponentConfig`
studio-field extension lands in **this** repo.

## 14. Suggested v1 build order

1. Project schema + `ProjectRoot` + component map; render one hardcoded project end-to-end
   (prove preview === render).
2. Generalize the render route + server-side validation.
3. better-auth + Drizzle + projects CRUD + dashboard + middleware redirect.
4. Editor shell: palette → timeline (BG/Scene/Overlay), clip add/move/resize, inspector from config.
5. Custom transition engine + seam UI.
6. Drag-on-canvas overlay handles.
7. `config.ts` → registry payload (this repo); `sync-registry` + studio-field retrofit on the
   curated set; thumbnails.
8. Polish, soft caps, watermark default, share links.

## 15. Open items / risks

- **`config.ts` → registry payload** is a prerequisite in this repo; without it the dogfood install
  can't carry studio metadata.
- **Preview perf** for stacked three.js scenes (`volumetric-rays`, `ai-generation-canvas`) in a live
  Player — plan a "preview quality" toggle.
- **Registry drift:** the studio pins a component set; re-sync is manual and deliberate.
- **Remotion Company License** must be secured before public launch (commercial use of Player +
  renderer).
- **Render queue:** one box, low `RENDER_MAX_CONCURRENT` — free beta + accounts means soft per-user
  caps must land before launch.

## 16. Deferred to v2

- Code export (readable `.tsx` composition + `npx shadcn add` list + JSON sidecar for round-trip).
- User uploads (images, then audio track, then video).
- Boundary transitions beyond the custom cross-fade set.
- Paid tiers (render limits / resolution / watermark / premium components).
