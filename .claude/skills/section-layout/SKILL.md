---
name: section-layout
description: Apply this landing page's section geometry rules — the shared left/right alignment rail, "fill the viewport without stretching content", and the first-card-visible budget for card grids. Use when adding a new section, editing an existing section's wrapper/padding/width, adding a nav anchor, or changing a card grid's columns, and when reviewing whether a change breaks alignment.
---

> Companion skill: **`vertical-rail-alignment`** covers alignment *within* a
> section — how a title row's left and right ends relate, and how two columns
> line up against the rail's two edges. This skill defines where the rail is;
> that one defines how content hangs off it.

# Section layout & alignment

Three rules govern where every section's content sits. They exist because the
page previously had four competing centring mechanisms that produced three
different left edges, and because sections sized to their content left partial
bands of the next section visible when landing on a nav anchor.

## Rule 1 — one shared rail

Every section's content aligns to a single left and right edge. That includes
section titles, the hero portrait, the about portrait, and the first/last card
of the videos and pictures grids.

The mechanism is `--rail-inset` in `input.css`, applied via `.rail`:

```css
--rail-inset: max(1.5rem, calc((100vw - 80rem) / 2 + 1.5rem));
.rail { width: 100%; padding-inline: var(--rail-inset); }
```

**Why viewport-relative and not `max-w-7xl mx-auto`.** The hero's content column
is only 80% of the viewport — the pink aside owns the other 20%. A
parent-relative cap centres inside whatever box it's in, so the hero landed on a
different edge than every other section, and the gap *widened* as the window
narrowed (192px vs 344px at 1920; 48px vs 127px at 1486). Keying to `100vw`
makes the same rule resolve to the same pixel edge in both contexts.

The formula deliberately reproduces the old `max-w-7xl` + `px-6` box exactly
(344px at 1920, clamping to a 24px floor at ≤1328px), so sections that were
already correct did not move.

### Applying it

- Section wrapper: `<div class="rail section-pad">`.
- `.section-pad` is **vertical padding only**. It used to carry `px-6`; horizontal
  padding now lives solely in `.rail`. A section using `.section-pad` without
  `.rail` will sit flush against the viewport edge.
- **Never** put `mx-auto` or a `max-w-*` cap on anything inside `.rail`. That
  re-centres content in a narrower box and reintroduces the misalignment. This
  is exactly what `max-w-5xl mx-auto` on the card grids was doing (+104px) and
  what `mx-auto` on the about portrait was doing (+108px).
- Images that need centring on mobile use `mx-auto md:mx-0` — centred while
  stacked, flush to the rail from `md` up. Decorative glows positioned behind
  such an image must track it (`m-auto md:mx-0`) or they drift off.
- Use **`ring-*`, not `border-*`**, for photo frames. A border is drawn inside
  the box, insetting the visible photo past the rail by the border width; a ring
  draws outside it. The hero portrait uses `ring-2` for this reason.

### Intentional exception

`contact.html` is a deliberately narrow centred CTA (`max-w-3xl mx-auto` +
`text-center`), not a rail-aligned section. It needs its own explicit `px-6`
because `.section-pad` no longer supplies horizontal padding. Don't "fix" it.

## Rule 2 — fill the viewport, don't stretch the content

`.section-fill` makes the section a flex column with a **min-height** of one
viewport minus the header and divider, then lets the natural-size content block
sit at the top of whatever height results:

```css
@media (min-width: 768px) and (min-height: 640px) {
  .section-fill {
    min-height: calc(100svh - var(--header-h) - var(--divider-h));
    display: flex;
    flex-direction: column;
    justify-content: flex-start;   /* top-anchored by default */
  }
  .section-fill-center { justify-content: center; }   /* opt-in; hero only */
}
```

The surplus becomes whitespace. Type sizes, image dimensions, gaps and aspect
ratios are **untouched** — a 1080p monitor shows the same layout as a laptop
with more air around it. Never scale content up to fill space.

- `--header-h` (4rem) matches the fixed header; `--divider-h` (4px) matches the
  inter-section rules. `main` carries `pt-16` as *padding*, so a full-height
  child does not overflow by the header height. Keep all three in sync.
- `svh`, not `vh`, so collapsing mobile browser chrome doesn't cause a jump.
- Applied to **hero, about, videos, pictures**. Analytics and contact size to
  their content — contact especially would sit in a large empty red field.
- **`.section-fill-relax` opts a section out of the fill on large desktops**
  (`min-width: 1920px`). Hero and about carry it: their content is a compact
  poster (~450–600px), so filling a 1080p+ viewport leaves a huge empty band.
  On ≥1920px (Full HD and up) they size to their content instead; below that
  they still fill. The selector is compounded (`.section-fill.section-fill-relax`,
  specificity 0,2,0) so it beats `.section-fill` regardless of @media ordering.
  Videos/pictures do NOT carry it — their card grids have the content to fill.
- Inert below `md` and below 640px viewport height: forcing full-height sections
  on a phone creates tall empty gaps and makes the page scroll forever.

**Content is top-anchored, not centred.** `justify-content: flex-start` is the
default; `.section-fill-center` opts into centring. Centring every section was
the first attempt and it read as broken at 1080p — a ~500px content block in a
1012px box pushed the heading ~350px down, so clicking a nav item showed an
empty colour band with a floating title. Top-anchoring puts the title just under
the header where the reader expects it. Only the hero uses
`.section-fill-center`, because it genuinely is a centred poster composition
rather than a titled section.

Anchor landing depends on `html { scroll-padding-top: calc(4rem + 4px) }` in
`@layer base` — keep it consistent with `--header-h` + `--divider-h`.

## Rule 3 — first card row must be fully visible

On any viewport, landing on `#videos` or `#pictures` must show the first row's
image **and** its caption (title, date, tag) without scrolling.

Two mechanisms cooperate:

- `.preview-media` is a fixed **1:1 square at every viewport** (rounded corners
  via `rounded-preview-sm`). An earlier version flattened it toward 4:3 / 16:9
  as viewport *height* shrank to fit a row on a short window, but that turned
  the thumbnails into rectangles on laptops / tablet-landscape and read as
  inconsistent next to the square hero/about portraits. The short-window case
  is now handled by the grid steps + natural scroll, not by distorting the
  thumbnail.
- Grids step `1 → 2 → 3` columns at `sm`/`lg` and cap there — six items form a
  deliberate 3×2 grid on desktop. (A 4-column `xl:` step was tried to keep
  cards narrow, but 3×2 is the chosen composition. Three columns do widen cards
  (~341px → ~410px at 1920) and so tall-en the 1:1 thumbnails, which pressures
  the first-card-visible budget; that's now held by the rail's content cap and
  `.section-fill-relax` on large desktops, not by a 4th column.)
- The `sm:` step matters independently: without it, landscape phones and small
  tablets (480–767px) render one full-width card per row.

Both grids must use identical column and gap rules, or their first cards won't
align with each other.

## Adding a new section — checklist

1. `<section id="..." class="bg-... section-fill">` (omit `.section-fill` if it
   should size to its content).
2. Inner wrapper: `<div class="rail section-pad">`.
3. Title uses `.section-title-pipe` with the pipe colour set per-section
   (`text-brand-red` on light backgrounds, `text-white` on pink/dark).
4. No `mx-auto` / `max-w-*` on anything inside the rail.
5. Add the nav entry to the `nav` param and confirm the anchor `id` matches —
   note the pictures section is `id="pictures"` while the partial is
   `gallery.html`.
6. Re-pick the adjacent divider colours in `index.html`: each 4px rule is chosen
   to contrast **both** neighbouring sections, so inserting a section means
   revisiting the two rules around it, not copying one.
7. Recompile the CSS — editing `input.css` has no effect until the Tailwind CLI
   regenerates `static/css/main.css`.

## Verifying a change

Measured values that should hold (left edge of every title and leading image):

| Viewport | Left edge | Fill height |
|---|---|---|
| 1920×1080 | 344px | 1012px |
| 1440×900 | 104px | 832px |
| 1024×768 | 24px | 700px (grids may exceed — `min-height`, so that's correct) |

Check all of: one distinct left edge across sections; hero/about/videos/pictures
at the fill height; first card's caption bottom inside the viewport after
anchor-scrolling; and `document.documentElement.scrollWidth` equal to the
viewport width (no horizontal overflow).

Hugo isn't required to check geometry — layout is a function of class names and
CSS. Rendering the partials with placeholder content and measuring
`getBoundingClientRect()` in headless Chrome is sufficient and catches real
regressions.

## Gotcha when editing `input.css`

Both mistakes made while building this were the same one: appending prose *after*
a comment block's closing `*/`, leaving text as bare CSS. The Tailwind CLI
reports it as `CssSyntaxError: Unterminated string` pointing at an apostrophe
several lines away from the real problem. When extending an existing comment,
merge into the block rather than adding after the delimiter — and compile before
declaring done.

## Changelog — what changed when these rules were introduced

The three rules above were retrofitted onto a page that already existed. This is
the full list of edits, so a reader can tell which classes are new machinery and
which are pre-existing.

### `src/input.css`

| Change | Reason |
|---|---|
| Added `:root` vars `--rail-inset`, `--header-h`, `--divider-h` | Single source of truth for the rail and the two heights `.section-fill` subtracts |
| Added `.rail` | Rule 1 — the shared left/right edge |
| Added `.section-fill` and `.section-fill-center` | Rule 2 — fill without stretching; centring is opt-in |
| `.section-pad`: `py-6 px-6 md:py-10` → `py-6 md:py-10` | Horizontal padding moved into `.rail`; keeping both double-padded rail sections |

`src/inputs.css` is a stray byte-identical duplicate and was left alone —
`input.css` is the file the build recipe names.

### Partials

| File | Change | Reason |
|---|---|---|
| `hero.html` | Added `section-fill section-fill-center`; removed the inert `flex-1` | `baseof.html` establishes no flex chain, so `flex-1` never did anything — hero rendered 645px in a 783px viewport |
| `hero.html` | Left column: added `rail`, removed `px-5 sm:px-8 md:px-12` and `justify-center`; inner wrapper lost `max-w-6xl` | Those were the parent-relative caps that put the hero portrait on a different edge (192px vs 344px at 1920) |
| `hero.html` | Portrait frame `border-2` → `ring-2` | A border draws *inside* the box, insetting the photo 2px past the rail; measured 346px instead of 344px |
| `about..html` | Added `section-fill`; `max-w-7xl mx-auto` → `rail` | Rules 1 and 2 |
| `about..html` | Portrait `mx-auto` → `mx-auto md:mx-0`, glow `m-auto` → `m-auto md:mx-0` | Unconditional `mx-auto` floated the portrait 108px inside the rail on desktop; the glow has to track it |
| `videos.html`, `gallery.html` | Added `section-fill`; `max-w-7xl mx-auto section-pad` → `rail section-pad` | Rules 1 and 2 |
| `videos.html`, `gallery.html` | Grids: removed `max-w-5xl mx-auto`, added `xl:grid-cols-4` | The inner cap re-centred the grid inside an already-centred box (+104px). The 4th column stops the now-full-width grid from inflating cards ~341px → ~410px, which would push captions off screen |
| `analytics.html` | `max-w-7xl mx-auto` → `rail`. No `section-fill` | Rail-aligned, but its stats are compact — a forced full height leaves a dark empty band |
| `contact.html` | Added explicit `px-6`, kept `max-w-3xl mx-auto` | Deliberately not rail-aligned; needs its own horizontal padding now that `.section-pad` is vertical-only |
| `footer.html` | Removed `px-6` from `<footer>`; `max-w-7xl mx-auto` → `rail` | So the footer's hairline rule ends on the same edge as the content above it |
| `header.html` | No geometry change | It keeps its own `max-w-[1600px]` cap: a full-bleed fixed bar, intentionally not on the content rail |

### Documentation pass

Every partial plus `index.html` and `_default/baseof.html` gained a banner
comment (role / params read / layout contract / internal sections / traps) and
numbered internal dividers. `index.html` also carries a page-flow map and a
per-divider note explaining each colour choice. `input.css` was reorganised into
8 numbered sections with `[Layout rule N of 3]` markers on `.rail`,
`.section-fill` and `.preview-media`. One stale comment was corrected: it still
described `.section-fill` as centring its content after the switch to
top-anchored.

### Rejected approaches, recorded so they aren't retried

- **`max-w-7xl mx-auto` everywhere instead of a viewport-relative rail.** Fails
  inside the hero, whose content column is 80% of the viewport — a
  parent-relative cap centres in that narrower box, and the discrepancy *grows*
  as the window narrows.
- **Centring content in filled sections.** Only caught by screenshot: a ~500px
  block in a 1012px box pushed the heading ~350px down, so clicking a nav item
  showed an empty colour band with a floating title. Now top-anchored, with
  `.section-fill-center` reserved for the hero.
- **Full-width grid at 3 columns.** Cards inflate, thumbnails get taller,
  captions drop below the fold — the exact failure Rule 3 exists to prevent.
