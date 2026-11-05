# Patches to taffy in this fork

Fork: `https://github.com/huangdonghai/taffy`, branch `fragmentation-basis`, taken at `ea07c4c` —
exactly the revision `deps/blitz` pins, so switching to this checkout changed no line of taffy's source
and the corpus read identically before any patch was written (5061 of 8276 pairs, 79 registered
differences, none stale, no pair broken).

## Why this fork exists

> **Read this first: the premise below was measured false, and the fork carries no source patch.**
> A percentage block size inside a fragmentainer does *not* resolve against that fragmentainer's depth;
> for a columned box it resolves against the whole container, as `css-multicol`'s own tests state, and
> the pair quoted below as the reason for this branch was failing on a `box-sizing` error in
> `deps/blitz` instead. See "What happened next" at the end. Everything between here and there is kept
> as the record of a hypothesis, not as an open work order.

Inside a fragmentainer — a column, a page — a percentage block size has to resolve against the
**fragmentainer's depth**, while the box being fragmented keeps its **own height**. In a `columns: 2` box
of 200px, a child of `height: 50%` is 50px: 50% of the 100px column, not of the 200px container
(`css-multicol/multicol-span-all-children-height-001`, whose WPT reference writes the child as `25%`, and
whose own markup says "Spread evenly into two columns, each 25%").

taffy cannot express that today, because one field does both jobs. `compute/block.rs` derives the value
children resolve percentages against from the box's own known size, gated by its definiteness:

```rust
let percentage_basis_dimensions = Size {
    width: known_dimensions.width,
    height: known_dimensions.height.filter(|_| inputs.known_dimensions_are_definite.height),
};
```

and the container's own height is that same `known_dimensions.height`. So setting it to the column depth
limits how deep the flow may be measured, which is wrong for a single measure of a 200px flow split into
two 100px columns ✓ measured: three acceptance pairs each got 5–10× worse. Setting a child's
`parent_size.height` instead does nothing at all for a block child, because the block algorithm replaces
that field when it builds the child's inputs (`parent_size = Size { width: Some(container_inner_width),
height: container_percentage_resolution_height }`) ✓ measured: byte-identical pixels on all three pairs.
Both results are the same fact seen from either side of the coupling.

## The change to make (not yet made)

A basis of its own, on the input the parent passes down:

1. **`src/tree/layout.rs`** — add to `LayoutInput` (next to `known_dimensions_are_definite`, ~line 153):

   ```rust
   /// When set, the block size a child's percentage block size resolves against, in preference to
   /// the box's own known height. For a box that is being fragmented: the depth of one fragmentainer.
   pub percentage_block_basis: Option<f32>,
   ```

   Add the field to `LayoutInput::HIDDEN` (`None`) and to `transpose()` — **kept unchanged** there, since
   the value is already the block-axis measure and the caller (blitz's multi-column pass) sets it in the
   writing mode it is fragmenting in. Document that, because a future `sideways-*` reader will otherwise
   assume it swaps.

2. **`src/compute/block.rs`** — at the `percentage_basis_dimensions` above, use
   `inputs.percentage_block_basis.or(known_dimensions.height.filter(…))`. Nothing else: the container's
   own height, the available space and the child positions stay as they are, which is what makes this
   different from the two failed attempts measured above.

3. **`src/tree/cache.rs`** — **the load-bearing part.** The measured-layout cache key is built from the
   input (`KnownDimensions`, `SizingMode`, …). The new field must be part of that key or a box measured
   once at one depth will be handed back inside a fragmentainer of another depth, which is a stale-layout
   bug that looks like a rounding error. Take it as `Option<f32>` (taffy already hashes floats in the key —
   follow what it does for `known_dimensions`) and add a test that measures the same node twice with
   different bases and gets different results.

4. Every other `LayoutInput { … }` literal in this crate (block/flex/grid/oof compute paths, and the
   trait defaults in `tree/traits.rs`) needs the field. `LayoutInput` does not derive `Default`, so the
   compiler lists them; it is mechanical, but do not paper over any of them with a `unwrap`-style default
   that silently differs from `None`.

5. **`deps/blitz/packages/blitz-dom/src/layout/multicol.rs`** — in the `narrow` input, set it to the
   depth one column has when the container's height is definite and the fill is balanced:
   `(definite − block insets) / column-count`. Leave it `None` for `column-fill: auto` (its fragments are
   as deep as the container ✓ the corpus there has not been re-read under this rule) and for an
   indefinite height, where today's answer is already the container's own content height ✓ spanners are
   unaffected: their fragmentainer *is* the container, and `remeasure_spanners` already gives them
   `parent_size.height` explicitly.

## Acceptance

- ~~`css-multicol/multicol-span-all-children-height-001` (209 px), `-002` (39720 px), `-003` (51537 px)
  — these three are the pairs the change is *for*.~~ **Retired: see "What happened next" below.** These
  pairs needed no percentage-basis change, and this fork has no part in them.
- ~~The pin `cuotuo-gpui probe::percentage_heights_in_a_column_ask_for_the_container` **must fail** ✓~~
  It now asserts the rule the WPT references give (a child's percentage asks for the whole multicol) and
  passes; it was rewritten rather than left as a marker for a change that turned out to be wrong.
- Gates, in this order: `cargo test -p blitz-dom`, `cargo test -p cuotuo-gpui`,
  `_scratch/bulletscore.ps1 -Diff` (eight families in ~40 s), then the full corpus
  (`conformance_score -- --ignored`) against the archived score at the same revisions.

## What happened next — the premise was wrong, and this fork is not needed for it

The change above **was** written, measured, on this branch, and reverted: adding
`percentage_block_basis`, reading it at `container_percentage_resolution_height` and carrying it in the
`CacheKey` compiled cleanly (ten literal sites in this crate, one in `blitz-dom`'s `remeasure_spanners`)
and the plumbing demonstrably worked — an unrelated pair (`…children-height-007`) moved by exactly the
4106px a changed children's basis moves ✓ so the value did reach the children. But `-001`, `-002` and
`-003` came back **byte-identical** (209 / 39720 / 51537 px). That was read as "the channel is elsewhere,
try `stylo_taffy::wrapper`". It was not: **there was no basis to change.**

- `-002` and `-003` say it in their own `assert`: the percentage is computed "relative to the entire
  multicol", and their references write the children as `height: 200px` in a container whose content is
  200px and whose columns are 100px deep — the child is spread over both columns, not one. Today's
  container basis is the rule. The reading that sent seven rounds here came from `-001`, whose children
  are `50%` against a reference `25%`: true, but `-001`'s column depth happens to be exactly half its
  container, so the pair cannot tell the two rules apart. The pin used `columns: 1`, where they coincide
  outright, and so proved nothing while appearing to pin something.
- What `-002`/`-003` were actually failing on was in `deps/blitz`, and is now fixed there: the authored
  `height` is the **content** box under the default `box-sizing`, while the block algorithm is handed a
  **border box** (the width in the same call already was), so a container of `height: 200px;
  border: 5px` measured its own content at 190px, its columns at 95, and every percentage inside at 95
  instead of 100. Re-basing the percentage could not show up as a difference while the basis itself was
  off by the border — which is the real reason this fork's field looked inert, and the reason the
  `parent_size.height` attempt before it looked inert too.

So: **no fifth-fork work is required for the percentage pairs, and none for anything else currently
open** (float and abspos across pieces, the nested vertical axis, vertical columns — those need the flow
measured per fragmentainer, which is a `blitz-dom` architecture change, not a taffy one). The open
question for whoever reads this next is whether this fork should exist at all; the honest answer today is
that its `percentage_block_basis` field was the only reason to pin it, that field is reverted, and the
decision belongs above this file.
- The next lead, and the reason both this and the `parent_size` attempt were inert for heights: the
  percentage is probably resolved before taffy ever sees it, in the style wrapper. `stylo_taffy::wrapper`
  hands taffy a `LengthPercentage` derived from stylo's computed values, and it already carries its own
  `percent_basis` (used for padding only ✓ `fn padding()`). Look for where a computed `height` percentage
  is turned into a resolved length in that wrapper — if it resolves against the container's height at
  that point, no `LayoutInput` field downstream can override it, and the basis has to be passed into the
  style wrapper instead (which is a `deps/blitz` change, not a taffy one ✓ and would also explain why the
  field behaves for padding and not for height).
- The corpus at this revision is worth 5061 pairs; a percentage-basis change reaches every box with a
  percentage height **inside any columned or paged container**, which is a wide blast radius in a
  direction the corpus has not been probed from. Treat `-002` and `-003` as *separate* mechanisms if they
  do not move with `-001`: `-002` is a border-box basis (5px border), `-003` a `300%` child whose spill
  needs the per-fragmentainer measure this field alone does not give.
- This field does **not** unlock float carrying, the flex row gap truncated at a line break, or the
  nested writing mode. Those need the flow measured once per fragmentainer, which is a different change
  and is written up in `deps/blitz/packages/blitz-dom/src/layout/fragmentation.rs`'s module docs.
