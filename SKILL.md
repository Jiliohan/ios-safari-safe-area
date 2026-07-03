---
name: ios-safari-safe-area
description: >-
  Fix iOS Safari (including iOS 26 "Liquid Glass") status-bar and safe-area
  problems on web pages: page content showing THROUGH / behind the translucent
  status bar (behind the clock/battery) when scrolling, white bars at the top or
  bottom behind Safari's toolbar, or content bleeding above a fixed header. Use
  this whenever someone reports that a mobile web page's top or bottom strip is
  "see-through", "transparent behind the status bar", "shows white behind the
  Safari buttons", "content scrolls behind the notch", or that a fixed
  header/nav doesn't cover the status bar on iPhone — even if they don't name
  iOS, Safari, or "safe area" explicitly. Also use for viewport-fit=cover,
  env(safe-area-inset-*), theme-color, notch, home-indicator, and 100dvh
  mobile-viewport questions.
---

# Fixing iOS Safari status-bar / safe-area bleed

## The symptom

On iPhone Safari, on a normal **scrolling** page:

- As you scroll, page content (headings, images, text) is visible **behind the
  translucent status bar** — the strip with the clock, Dynamic Island, signal,
  and battery. The top strip looks "see-through".
- Or there's a **white band** at the very top and/or bottom (behind Safari's
  floating toolbar buttons) that you can't color.
- A `position: fixed` header with a solid background **does not** cover the
  status-bar strip, no matter how opaque you make it.

This got dramatically worse in **iOS 26 ("Liquid Glass")**, where the browser
chrome became translucent and content-tinted.

## Why it happens (the root cause — this is the key insight)

**iOS Safari composites the *scrolling document's* pixels behind the translucent
status bar, and `position: fixed` elements do NOT count.**

So while the document (root scroller) is scrolling, Safari paints the moving
page content up into the status-bar area. You can stack an opaque fixed header
at `top: 0` with `padding-top: env(safe-area-inset-top)` all you want — Safari
ignores it for that strip and keeps showing the scrolling content. This is why
"just make the nav opaque" never works.

Two more facts that trip people up:

- **iOS 26 ignores `theme-color`** for the chrome tint (a WebKit regression;
  reportedly targeted for a later 26.x). Safari instead tints the top/bottom
  strips from the **`html`/`body` background-color**.
- At `scrollY === 0`, Safari uses the root background color; **once the document
  has non-zero scroll, it composites real page pixels** behind the status area.
  That "once scrolled" behavior is exactly the bleed.

The reason a **non-scrolling** page (a single-screen landing that fits in
`100dvh`) looks perfect is that there is no scrolling document to composite —
the strip just shows the solid body color. That is the whole trick, generalized
below.

## The fix: lock the document, scroll an inner container (app-shell)

If the document itself never scrolls, Safari has no scrolling pixels to
composite behind the status bar, so the strip stays the **solid body
background color**. Move all scrolling into an inner element instead.

This is the reliable fix for a long, scrolling page. Everything else
(opaque overlays, `theme-color`, taller headers, tweaking the nav) is a dead end
on iOS 26.

### 1. Lock the document, make `#scroll-root` the scroller (`globals.css`)

```css
/* App-shell: the document itself does NOT scroll — an inner #scroll-root does.
   iOS Safari composites the scrolling document's pixels behind the status bar,
   so locking the document keeps that top strip the solid body color. */
html,
body {
  height: 100dvh;
  overflow: hidden;
  overscroll-behavior: none; /* no rubber-band revealing the edge */
}

body {
  background-color: #fafafa; /* this color is what the safe-area strips show */
  color: #0a0a0a;
}

#scroll-root {
  height: 100dvh;
  overflow-y: auto;
  overflow-x: clip;
  overscroll-behavior: none;
  scroll-behavior: smooth;
  -webkit-overflow-scrolling: touch; /* momentum scroll on iOS */
}
```

### 2. Wrap the content (Next.js App Router `layout.tsx`)

Keep the fixed nav **outside** the scroller (it stays viewport-fixed), and put
everything scrollable inside `#scroll-root`.

```tsx
<body className="font-sans" suppressHydrationWarning>
  <Navbar />                       {/* position: fixed — stays put */}
  <div id="scroll-root">
    {children}
    <Footer />
  </div>
  <Analytics />
</body>
```

Also keep the viewport meta with cover — you still want to paint edge-to-edge:

```ts
// export const viewport (App Router)
export const viewport = {
  viewportFit: "cover",
  themeColor: "#fafafa", // harmless; still honored by Chrome/Android
};
```

### 3. Point scroll listeners at `#scroll-root`, not `window`

`window.scrollY` is now always `0`. Read the inner container's `scrollTop`
instead. Attach on **`document` in the capture phase** — scroll events don't
bubble, but they *do* run capture, so this catches `#scroll-root`'s scroll
regardless of component mount order (no fragile `getElementById` race).

```tsx
useEffect(() => {
  const onScroll = () => {
    const scroller = document.getElementById("scroll-root");
    const y = scroller ? scroller.scrollTop : window.scrollY;
    // ...your scroll-based state here (e.g. nav dark→light past the hero)...
  };
  onScroll();
  document.addEventListener("scroll", onScroll, { capture: true, passive: true });
  window.addEventListener("resize", onScroll);
  return () => {
    document.removeEventListener("scroll", onScroll, { capture: true });
    window.removeEventListener("resize", onScroll);
  };
}, [pathname]);
```

## What still works unchanged (verify these after applying)

- **Hash navigation** (`href="/#about"`): the browser and `element.scrollIntoView()`
  scroll the nearest scrollable ancestor, which is now `#scroll-root`. Works.
- **IntersectionObserver / framer-motion `whileInView` scroll-reveals**: fire
  normally. IO uses the **viewport** as root by default; elements moving inside
  `#scroll-root` still change their viewport intersection, so reveals trigger.
- **Momentum scrolling / smooth scroll**: preserved via the `#scroll-root` CSS
  above.

## `viewport-fit=cover` — keep it, understand the tradeoff

`viewport-fit=cover` is what lets the page paint **behind** the notch and the
bottom toolbar (so the bottom edge shows your color instead of Safari's white
chrome). Keep it. Its downside — letting content scroll behind the status bar —
is exactly what the locked document fixes. So `cover` + app-shell together give
you: full-bleed edges **and** a solid status-bar strip.

If you ever ship *without* the app-shell, note the fork:
- **`cover` on, document scrolls** → bottom fills with your color, but content
  bleeds behind the top status bar. (The broken state.)
- **`cover` off** → top strip is a solid body-color (content can't reach it),
  but the bottom behind the toolbar reverts to Safari's own light chrome.

You can't win both with a plain scrolling document. That's why the app-shell
exists.

## Coloring the strips (optional)

The top/bottom safe-area strips render the **`body` background-color**. Pick the
`body` background you want those strips to be. If you need the strip color to
change with scroll position (e.g. dark over a dark hero, light elsewhere),
prefer a **static** body color first and confirm the bleed is gone; dynamically
mutating `document.body.style.backgroundColor` on scroll is unreliable for the
iOS chrome tint and should be treated as a nice-to-have, not the fix.

## When NOT to reach for the app-shell

- The page already **fits in one screen** and doesn't scroll → just use
  `viewport-fit=cover` + a solid `body` background; you're done.
- You control a native shell / PWA `display: standalone` → different rules
  (`apple-mobile-web-app-status-bar-style`), out of scope here.

## Quick verification checklist (on a real iPhone)

1. Scroll into a light section — the strip behind the clock/battery stays a
   solid color, no content showing through.
2. The very bottom behind Safari's buttons shows your color, not white.
3. Nav/scroll-based UI still updates while scrolling.
4. Tapping in-page anchors (`#about`) still jumps to the section.
5. Scroll-reveal animations still play.

The desktop browser and headless Chrome **cannot** reproduce iOS safe-area
behavior, so always confirm on a device (or at least a real iPhone Safari), not
just a resized desktop viewport.
