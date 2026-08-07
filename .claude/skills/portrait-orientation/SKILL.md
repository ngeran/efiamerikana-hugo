# Portrait-orientation layout

**Status: APPLIED** (was "validated in a harness, not applied"). The two
defects below are fixed in `src/input.css` (gate clause + portrait block) and
`partial/hero.html` (`hero-split` class). Landscape, tablet-landscape, and
mobile are unchanged — see Constraints.

Companion to **`section-layout`** (the three *landscape* geometry rules) and
**`vertical-rail-alignment`** (in-section alignment, width-driven, unaffected).
This skill exists because two landscape rules leaked into portrait: the page was
tuned wide-and-short, and `min-width: 768px` does **not** mean "landscape" — a
tablet held portrait is 768–1024px *wide* and passes `md`.

## Defect 1 — sections fill to a height their content cannot use

`.section-fill` (section-layout rule 2) gates on
`@media (min-width: 768px) and (min-height: 640px)`. A rotated 1080×1920
monitor passes both, so `#about` got a 1852px floor for 543px of content. In
portrait, `100svh` grows while the content — sized by *width* — does not, so the
trailing gap crosses ~50% and reads as a broken section. Measured baseline:

| Viewport | Section | Content | Trailing gap | Wasted |
|---|---|---|---|---|
| 768×1024 | about | 497px | 459px | 48% |
| 834×1112 | about | 500px | 545px | 52% |
| 1024×1366 | about | 543px | 755px | 58% |
| 1080×1920 | about | 543px | **1309px** | **71%** |
| 1080×1920 | videos | 1012px | 840px | 45% |

In landscape the surplus is 25–45% and reads as deliberate breathing room.

**Fix:** add `and (orientation: landscape)` to the gate. In portrait every
section then sizes to its content — the *existing* mobile behaviour extended to
tall viewports, not a new layout, and explicitly **not** a guessed portrait
`min-height` (the content is already the right height; `.section-fill` exists
only so a short *wide* window doesn't reveal the next section's colour band, a
failure mode that does not exist in portrait).

## Defect 2 — hero aside overflows the page at 768–805px wide

The hero is an 80/20 split (`md:w-4/5` + `md:w-1/5`). At 768px the aside is
153.6px; after its 32px left border and 2×24px padding the inner box is
**73.6px**, while the "view my portfolio" `<h4>` is at `md:text-4xl` (36px). It
cannot fit: `scrollWidth` is 773 against a 768 viewport — a 5px horizontal
scroll. Range **768–~805px, portrait only** (at 767 the hero is already stacked
below `md`; by ~810 the aside is wide enough).

**Fix:** stack the hero in portrait — `.hero-split { flex-direction: column }`
and `.hero-aside { width: 100%; border-left-width: 0 }`, in a new section 9.
The inner box goes 73.6px → ~720px and the overflow disappears. This mirrors
what the hero already does below `md`.

## The specificity trap — section 9 MUST live outside `@layer`

The portrait block is **unlayered** (top-level `@media`, not inside
`@layer components`). This is mandatory, not stylistic.

Tailwind utilities (`md:flex-row`, `md:w-1/5`) live in the `utilities` layer.
Layered CSS loses to a utility of equal-or-higher layer regardless of selector
specificity or source order. During validation an identical
`.hero-split { flex-direction: column }` placed **inside** `@layer components`
had **zero effect** — the hero stayed an 80/20 row and the overflow persisted at
exactly 773px. It looked like the media query wasn't matching; it was matching
and being overridden. Unlayered CSS beats all layered CSS, so the block sits at
the top level. **Do not "tidy" it into a layer** — the hero stacking will
silently break.

## Changelog (the applied fix)

| File | Change | Reason |
|---|---|---|
| `src/input.css` banner | Added `9. Portrait overrides` to the WHAT'S IN HERE list | New section must appear in the table of contents |
| `src/input.css` `.section-fill` gate | `@media (min-width:768px) and (min-height:640px)` → added `and (orientation: landscape)` | Defect 1 — stop the fill firing in portrait, where content (width-sized) can't use the floor |
| `src/input.css` `.section-fill` comment | Extended (not replaced) with the portrait rationale + the 1309px/71% measurement | House style: record *why* a clause exists and what it previously broke |
| `src/input.css` §9 (new, unlayered) | `@media (orientation: portrait) { .hero-split{flex-direction:column} .hero-aside{width:100%;border-left-width:0} }` | Defect 2 — stack the hero, full-width the aside, kill the 5px overflow |
| `partial/hero.html` `<section>` | Added `hero-split` class (kept `flex flex-col md:flex-row`) | Explicit portrait stacking hook; avoids `:has()` support question. The flex utilities still drive landscape + mobile |
| `partial/hero.html` banner | LAYOUT CONTRACT note: 80/20 split is landscape-only, `hero-split` is the portrait hook | So a future reader doesn't remove the stacking class or the flex utilities |

## Constraints — what must NOT change

1. **Landscape geometry is frozen.** The gate clause is a no-op in landscape
   (`orientation: landscape` still matches), and the portrait block doesn't
   match in landscape. No left edge, fill height, card-grid count, gap, or
   padding changes in landscape.
2. **Do not touch the height-based media queries** (`max-height: 820px` / `680px`
   on `.section-pad`, `.section-intro`, `.hero-pad`, `.preview-media`). They are
   orientation-neutral and correct — a genuinely shallow window is already a
   landscape condition; portrait is never <680px tall in practice.
3. **Do not "fix" mobile's multiple left edges** (430×932 → 24/55/256/292/294px).
   Pre-existing and intentional: `.rail` alignment is promised only from `md`
   up; mobile deliberately centres via `mx-auto md:mx-0`.
4. **Never hand-edit `static/css/main.css`**; edit only `src/input.css`
   (`src/inputs.css` is a stray duplicate — leave it).
5. **No new dependencies, no JS.**

## Verification

Mirror the partials into `layouts/partials/`, `baseof.html` into
`layouts/_default/`, render with placeholder content, and measure
`getBoundingClientRect()` in headless Chrome. Measure **trailing gap**
(`sectionBottom − deepestVisibleDescendantBottom`, skipping `aria-hidden`
ghosts/glows), not section height. Required portrait results: max trailing gap
≤1px and no overflow at 768×1024 / 834×1112 / 1024×1366 / 1080×1920. Landscape
trailing gaps at 1440×900 (273.2px, videos) and 1920×1080 (453.2px, videos) must
be **identical before and after**. Use `matchMedia` to label orientation — per
CSS spec `portrait` matches when height **≥** width, so 900×900 is portrait (do
**not** use `innerWidth < innerHeight`, which mislabels it).

### Open item — aside top-rule thickness in portrait tablet

The task spec expected the portrait aside's top red rule to render at **32px**
(supplied by `sm:border-t-[32px]` once the left rule is dropped). **Verified
against the compiled CSS:** `.md\:border-t-0` (top=0, in the `md` 768px query)
is emitted *after* `.sm\:border-t-[32px\]` (top=32px, in the `sm` 640px query),
so at **≥768px the top rule resolves to 0, not 32px** — the portrait override
only sets `border-left-width`/`width` and does not rescue it. The hero stacking
and the overflow fix are applied and CSS-verified; the top rule at 768px
portrait should be visually confirmed, and if a divider is wanted there, add
`border-top-width: 32px` to the portrait `.hero-aside` rule (the spec's "do not
hard-code" was premised on `sm:` supplying it, which `md:` overrides).
