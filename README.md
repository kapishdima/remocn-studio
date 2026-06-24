# Remocn studio

**A hosted, in-browser video editor that builds videos from remocn components.**

Drag components onto a timeline, configure them, preview live, and render an MP4 — all in the browser.

[Open Studio](https://studio.remocn.dev) · [remocn.dev](https://remocn.dev)

<p align="center">
  <img alt="header" src="https://shieldcn.dev/header/transparent.svg?mode=dark&amp;border=false&amp;image=https%3A%2F%2Fremocn.dev%2Fstudio.png&amp;overlay=0" />
</p>

![badge group](https://shieldcn.dev/group/github/kapishdima/remocn-studio/stars+github/kapishdima/remocn-studio/license+x/follow/kapish_dima.svg?variant=secondary)

## What it is

remocn studio turns the first-party [remocn](https://remocn.dev) component set into a video editor. You assemble a video from a known catalog of motion components, preview it instantly with the Remotion Player, and render a final MP4 on the server.

- **Drag, configure, render.** Pick components from the palette, drop them on the timeline, tune their props in the inspector, hit render.
- **Preview is the render.** The browser preview and the server-rendered MP4 run the exact same code — what you see is what you get.
- **No setup.** Hosted at `studio.remocn.dev`. Sign in and start editing.

## Features (v1)

- Typed 3-track timeline with drag / resize / snap
- Live preview via `@remotion/player`
- Property inspector generated from each component's config
- Drag-on-canvas placement for overlays
- Server-side MP4 rendering with a job queue
- Accounts, saved projects, and shareable links
