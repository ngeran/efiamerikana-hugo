---
name: vertical-rail-alignment
description: Align a section's content to the shared rail's left and right vertical edges — title rows with a right-hand sublabel, two-column portrait/copy or stats layouts, and card captions with a trailing tag. Covers which cross-axis alignment to pick (baseline / end / center / start), why right-edge items need shrink-0 and a max-w cap, and how stacked-mobile centring flips to column alignment at md. Use when adding or editing anything that has a left item and a right item inside one section.
---

# Vertical rail alignment

The companion skill **`section-layout`** defines *where* the rail's two vertical
edges are (`.rail` / `--rail-inset`). This skill covers what happens *at* those
edges: how a left item and a right item inside the same section relate to each
other and to the rail.

Two vertical lines run the whole page — the rail's left edge and its right edge.
Every section hangs content off one or both. Getting alignment right is mostly
about not accidentally introducing a *third* edge.

## The two edges

```
│                    rail left edge          rail right edge                    │
│                            │                            │                     │
│  ← --rail-inset →          │  content spans this box     │   ← --rail-inset →  │
```

Inside `.rail`, padding is symmetric. So a **full-width flex child with
`justify-between`** puts its first item exactly on the left edge and its last
item exactly on the right edge, with no extra classes. That single fact drives
every pattern below.

## Pattern 1 — title row: title left, sublabel right

Used by `videos.html` and `gallery.html`:

```html
<div class="flex flex-col md:flex-row justify-between items-end mb-8 gap-4">
  <div class="section-title-pipe">
    <span class="pipe text-brand-red">|</span>
    <span class="title-text">{{ .Param "videos_label" }}</span>
  </div>
  <p class="max-w-xs text-xs font-bold uppercase tracking-widest text-charcoal/40">…</p>
</div>
```

Three deliberate choices:

**`items-end`, not `items-center`.** The two items have wildly different type
sizes — a `clamp(1.25rem, 3vw, 1.75rem)` serif title against a `text-xs`
uppercase label. Centring them on the cross axis leaves the small label floating
in the middle of the title's box, visually unattached to anything. Ending them
sits the label's last line on the title's baseline region, so the two read as one
horizontal rule of information.

**`max-w-xs` on the right item.** Without a cap, a long sublabel grows leftward
until it collides with the title. The cap makes it wrap into a small block that
stays anchored to the right edge. Cap the *right* item, never the left — the left
item is the one that defines the section's identity and should be free to size to
its text.

**`flex-col md:flex-row` + `gap-4`.** While stacked there is no right edge to
speak of; the sublabel simply follows the title. `gap-4` covers the stacked case
(vertical gap) and the row case (minimum horizontal gap) with one class.

### Within the title itself: `items-baseline`

`.section-title-pipe` is `flex items-baseline gap-2`. The decorative `|` glyph
and the title text are different font sizes and different families, so `center`
would float the pipe. Baseline is correct whenever two pieces of *text* sit side
by side; end/center are for text-against-not-text.

### A title row with no sublabel

`about..html` keeps the same wrapper (`justify-between items-end`) even though it
holds only the title. `justify-between` and `items-end` are inert with one child.
Keep them anyway: it means adding a sublabel later is a one-line change, and
every section's title row is structurally identical, so a reader who has read one
has read them all.

## Pattern 2 — two columns hung off both edges

`about..html` (portrait | copy) and `analytics.html` (stats | bars):

```html
<div class="grid grid-cols-1 md:grid-cols-2 gap-8 items-center">
```

The grid spans the full rail, so column 1 starts on the left edge and column 2
ends on the right edge automatically. Do **not** add `max-w-*` or `mx-auto` to
the grid or to either column — that is the mistake that produced the page's
original three-different-left-edges problem.

**`items-center` here, not `items-end`.** The two columns are comparable blocks
of similar visual weight, and their heights differ by whatever the copy happens
to run to. Centring keeps them looking paired; top-aligning makes the shorter
column look like it fell off the top, and bottom-aligning makes it look like it
sank.

**Content inside a column still needs its own alignment.** `items-center` is the
*cross*-axis; it says nothing about where a narrower child sits horizontally
inside its column. That is why the about portrait carries `mx-auto md:mx-0` and
the copy carries `text-center md:text-left` — see Pattern 4.

**Right-column content aligns to the right edge only if it fills its column.**
`analytics.html`'s progress bars use `w-full` and their labels use
`flex justify-between`, so the platform name sits on the column's left and the
percentage on the rail's right edge. Its footnote deliberately uses `max-w-sm`
and so does *not* reach the right edge — that's fine for a de-emphasised note,
but don't do it to anything structural.

## Pattern 3 — card caption: title left, tag right

```html
<div class="flex justify-between items-start gap-2">
  <h3 class="font-serif text-xl font-bold leading-tight">{{ .Title }}</h3>
  <span class="shrink-0 text-[10px] … rounded-full">{{ . }}</span>
</div>
```

**`items-start`, not `items-end`.** Card titles wrap to two lines at some
viewport widths and not others. With `items-end` the pill would drop to the
second line's baseline on some cards and not others, so a row of cards would show
pills at inconsistent heights. `items-start` pins every pill to the top of its
caption, and the row stays visually level regardless of wrapping.

**`shrink-0` on the right item is mandatory.** Flex items shrink by default. A
pill or a "⏵ 1.2M Views" span that shrinks starts wrapping mid-word or squashing
its padding while the title happily takes the space. Every right-hand trailing
element in this codebase carries `shrink-0` for that reason.

This applies at card scale, not rail scale — the right edge here is the card's,
not the page's. But the rules are the same, which is the point: pick the
cross-axis alignment from *why* the two items differ, not from habit.

## Pattern 4 — stacked mobile centres, columns align

The recurring pair:

```html
class="mx-auto md:mx-0"          <!-- images -->
class="text-center md:text-left" <!-- copy   -->
```

Below `md` there is one column, and a centred block reads better than one hugging
the left edge with a ragged right. From `md` up the item is in a column paired
with another, and left-alignment is what makes them read as a pair.

Two traps:

- **Decorative layers must track the element they decorate.** The about
  portrait's blurred glow is `m-auto md:mx-0`, mirroring the photo's
  `mx-auto md:mx-0`. When it was left as unconditional `m-auto` it drifted toward
  the column's centre on desktop while the photo stayed on the rail, leaving a
  visible offset halo.
- **`mx-auto` on a full-width element does nothing; on a capped one it moves
  it.** The about portrait has `max-w-72/80/96`, so `mx-auto` shifted it 108px
  inside the rail. If you add a width cap to something, re-check its margins.

## Pattern 5 — sections that intentionally ignore the edges

Not everything belongs on the rail, and forcing it there is its own bug:

| Element | Alignment | Why |
|---|---|---|
| `contact.html` | `max-w-3xl mx-auto text-center` | A closing CTA reads as a full stop; symmetry beats edge-alignment. Needs its own `px-6` since `.section-pad` is vertical-only |
| `hero.html` right aside | Full-bleed, outside `.rail` | It's a colour block that must run to the viewport edge |
| `header.html` | Own `max-w-[1600px]` cap | A fixed full-bleed bar, not a content section |
| `analytics.html` ghost word | `absolute left-0` | Decorative, `aria-hidden`, deliberately bleeding past the rail |

If you add an exception, comment *why* in the partial. The default assumption a
reader makes is "everything is on the rail", so a silent exception looks like a
bug.

## Choosing the cross-axis alignment

| Situation | Use | Reason |
|---|---|---|
| Two runs of text, different sizes, side by side | `items-baseline` | Their baselines are the shared reference; anything else floats one of them |
| Large title vs small label, one row | `items-end` | Ends them together so the label attaches to the title instead of floating mid-box |
| Two comparable content blocks of unequal height | `items-center` | Keeps them reading as a pair; either extreme makes the shorter one look dropped |
| A wrapping title vs a fixed trailing chip | `items-start` | The chip stays level across a row of cards regardless of how each title wraps |

## Checklist for anything with a left item and a right item

1. Wrapper is `flex … justify-between` (or a full-width grid) with **no**
   `max-w-*` / `mx-auto` on the wrapper itself.
2. Cross-axis alignment picked from the table above, not copied blindly.
3. Right item has `shrink-0` if it must not compress, and a `max-w-*` if its text
   could grow toward the left item.
4. Stacked case handled: `flex-col md:flex-row` plus `gap-*`, and any centring
   scoped `md:`-up (`mx-auto md:mx-0`, `text-center md:text-left`).
5. Decorative layers behind an aligned element repeat that element's alignment
   classes.
6. Verify by measuring, not by eye: `getBoundingClientRect().left` of every
   left-hand item should be one value, and `.right` of every right-hand item
   another, across 1920×1080 / 1440×900 / 1024×768. Expected left edges are
   344 / 104 / 24px — see `section-layout`'s verification table.
