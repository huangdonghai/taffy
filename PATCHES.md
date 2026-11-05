# Patches to taffy in this fork

Fork: `https://github.com/huangdonghai/taffy`, branch `fragmentation-basis`, taken at `ea07c4c` —
exactly the revision `deps/blitz` pins, so switching to this checkout changed no line of taffy's source
and the corpus read identically before any patch was written (5061 of 8276 pairs, 79 registered
differences, none stale, no pair broken).

## Why this fork exists

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

- `css-multicol/multicol-span-all-children-height-001` (209 px), `-002` (39720 px), `-003` (51537 px) —
  these three are the pairs the change is *for*.
- The pin `cuotuo-gpui probe::percentage_heights_in_a_column_ask_for_the_container` **must fail** ✓ it
  asserts today's basis deliberately, and its comment says why: it exists to make this change visible
  rather than silent. Update it in the same commit, keeping its shape (assert the fragmentainer's depth,
  not the container's).
- Gates, in this order: `cargo test -p blitz-dom` (102), `cargo test -p cuotuo-gpui` (38),
  `_scratch/bulletscore.ps1 -Diff` (eight families in ~40 s), then the full corpus
  (`conformance_score -- --ignored`) against the archived score at the same revisions.

## Notes for whoever picks this up

- The corpus at this revision is worth 5061 pairs; a percentage-basis change reaches every box with a
  percentage height **inside any columned or paged container**, which is a wide blast radius in a
  direction the corpus has not been probed from. Treat `-002` and `-003` as *separate* mechanisms if they
  do not move with `-001`: `-002` is a border-box basis (5px border), `-003` a `300%` child whose spill
  needs the per-fragmentainer measure this field alone does not give.
- This field does **not** unlock float carrying, the flex row gap truncated at a line break, or the
  nested writing mode. Those need the flow measured once per fragmentainer, which is a different change
  and is written up in `deps/blitz/packages/blitz-dom/src/layout/fragmentation.rs`'s module docs.
