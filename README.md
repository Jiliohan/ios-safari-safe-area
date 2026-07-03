# ios-safari-safe-area

A [Claude](https://claude.com/claude-code) **skill** that captures the real fix
for iOS Safari (including iOS 26 "Liquid Glass") status-bar and safe-area bleed
on scrolling web pages.

> Page content shows **through the translucent status bar** (behind the
> clock/battery) when you scroll, or there's a **white bar** at the top/bottom
> behind Safari's toolbar, and no opaque fixed header fixes it.

## The one insight

**iOS Safari composites the _scrolling document's_ pixels behind the translucent
status bar — `position: fixed` elements do not count.** So you can't cover that
strip with an opaque nav while the document scrolls. (`theme-color` is also
ignored in iOS 26; Safari tints the strips from the `html`/`body` background.)

**Fix:** lock the document and scroll an inner container (an app-shell). With the
document static, the top strip stays the solid body-background color.

```css
html, body { height: 100dvh; overflow: hidden; overscroll-behavior: none; }
body { background-color: #fafafa; }        /* what the safe-area strips show */
#scroll-root { height: 100dvh; overflow-y: auto; -webkit-overflow-scrolling: touch; }
```

```html
<body>
  <nav>…</nav>                 <!-- position: fixed, stays viewport-fixed -->
  <div id="scroll-root">…</div> <!-- everything scrollable lives here -->
</body>
```

Full write-up, companion changes (scroll listeners, hash nav, IntersectionObserver
reveals), `viewport-fit=cover` tradeoffs, and copy-pasteable Next.js App Router
code are in [`SKILL.md`](./SKILL.md).

## Install (Claude Code)

Copy the skill into your personal skills folder:

```bash
git clone https://github.com/<you>/ios-safari-safe-area.git \
  ~/.claude/skills/ios-safari-safe-area
```

or, in an existing clone:

```bash
mkdir -p ~/.claude/skills/ios-safari-safe-area
cp SKILL.md ~/.claude/skills/ios-safari-safe-area/
```

Claude Code auto-discovers skills in `~/.claude/skills/`. It triggers whenever you
describe the symptom — even in plain words like "the top is see-through" or
"white behind the Safari buttons" — no need to name iOS or "safe area".

## Why this exists

Distilled from a long real-world debugging session getting a Next.js portfolio's
hero to render edge-to-edge on iPhone without content bleeding behind the status
bar. Written down so nobody has to rediscover it.

## License

MIT — see [`LICENSE`](./LICENSE).
