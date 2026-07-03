# ios-safari-safe-area

A [Claude Code](https://claude.com/claude-code) skill that fixes iOS Safari
(including iOS 26 "Liquid Glass") status-bar and safe-area bleed on scrolling web
pages.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)

## Install

```bash
npx skills add https://github.com/Jiliohan/ios-safari-safe-area
```

Or install it manually into your skills folder:

```bash
git clone https://github.com/Jiliohan/ios-safari-safe-area.git \
  ~/.claude/skills/ios-safari-safe-area
```

Claude Code auto-discovers skills in `~/.claude/skills/`.

## What It Does

Fixes the bug where, on a scrolling web page in iOS Safari, page content shows
**through the translucent status bar** (behind the clock and battery), or a
**white bar** appears at the top or bottom behind Safari's toolbar — and no
opaque fixed header covers it.

**Root cause.** iOS Safari composites the _scrolling document's_ pixels behind
the translucent status bar, and `position: fixed` elements do not count.
`theme-color` is ignored in iOS 26; Safari tints the strips from the
`html`/`body` background instead.

**Fix.** Lock the document and scroll an inner container (an app-shell). With the
document static, the top strip stays the solid body-background color.

```css
html, body { height: 100dvh; overflow: hidden; overscroll-behavior: none; }
body { background-color: #fafafa; }        /* the safe-area strips show this */
#scroll-root { height: 100dvh; overflow-y: auto; -webkit-overflow-scrolling: touch; }
```

```html
<body>
  <nav>…</nav>                  <!-- position: fixed, stays viewport-fixed -->
  <div id="scroll-root">…</div> <!-- everything scrollable lives here -->
</body>
```

## What's Included

The skill ([`SKILL.md`](./SKILL.md)) covers:

- The root cause, and why opaque fixed headers, `theme-color`, and taller nav
  bars do not work on iOS 26.
- The app-shell fix, with copy-pasteable Next.js App Router code (`globals.css`,
  `layout.tsx`, and the scroll listener).
- Companion changes: reading scroll from the inner container via a document
  capture-phase listener, hash navigation via `scrollIntoView`, and why
  framer-motion `whileInView` reveals still fire.
- `viewport-fit=cover` and the top/bottom safe-area tradeoffs.
- A real-device verification checklist.

## When It Triggers

The skill activates whenever you describe the symptom — even in plain words like
"the top is see-through", "white behind the Safari buttons", or "content scrolls
behind the notch" — without needing to name iOS, Safari, or "safe area".

## Compatibility

Works with any agent that reads `SKILL.md` skills from `~/.claude/skills/`
(Claude Code and compatible agents). The code examples target Next.js App Router,
but the technique is framework-agnostic.

## License

MIT — see [`LICENSE`](./LICENSE).
