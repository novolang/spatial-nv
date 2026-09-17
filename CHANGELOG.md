# Changelog

All notable changes to spatial-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `spatialrtree` and `spatialkdtree` — the load-bearing interface, and
  it is the same decision in both: an index holds an integer HANDLE
  into the caller's own list beside the box or the point, and never the
  caller's item. A generic index holding the items is what a language
  with working generics would publish; ndarray-nv 0.0.1 measured this
  toolchain and published the result — a generic container crossing a
  module boundary stops the compiler with E6000 or lowers its result as
  a pointer and fails IR verification — and a library is nothing but a
  module boundary. The handle earns its place beyond that: an item can
  be in three indexes without being stored three times, and the index's
  memory is proportional to the item count rather than to the item
  size. What the caller owes in exchange is stated as a rule in the
  README: reorder the list and the index is stale.
- Bulk loading and insertion are two named constructors rather than one
  with a flag, because they produce measurably different trees:
  Sort-Tile-Recursive packing fills every leaf and keeps sibling boxes
  from overlapping, and insertion does neither. `entries`, `items` and
  `depth` are published so a caller can see when a tree is worth
  rebuilding and can rebuild it.
- The tree nodes are genuinely recursive — an `RtreeNode` holding a
  list of itself, a `KdTreeNode` holding `?KdTreeNode` children. Both
  shapes were measured against this toolchain before they were written,
  across a module boundary and at run time, and both work, so the
  package publishes the honest structure rather than a flat arena with
  children by index.
- `spatialbox` — the algebra both trees are built out of, published
  rather than hidden: `enlargement` is the quantity an R-tree insertion
  minimises and `distance_to_point` is the quantity every
  nearest-neighbour search prunes on, so a caller can see what the tree
  optimises and measure it on its own data. Every two-box operation is
  fallible, because two boxes of different dimensions cannot be
  compared and the alternative to refusing is a confident answer about
  the wrong thing.
- `spatialquery` — four questions, not one with a flag. A range query
  takes a box and answers bare handles; a radius query takes a circle
  and answers hits with distances; nearest answers one or none;
  k-nearest answers at most k in increasing distance. A `SpatialHit`
  carries the distance the search already computed, because the caller
  would otherwise recompute it over items this package deliberately
  does not hold.
- `spatialfault` — eight refusals, every one about the CALL rather
  than about the data. An empty result is never one of them: an empty
  index, an area holding nothing and a k larger than the index all
  succeed.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  spatial-nv.<module>.<fn>`.
- **Every distance is planar.** A caller indexing longitude and
  latitude gets a correct planar index over degrees, which is not a
  ranking by true distance away from the equator, and a box crossing
  the antimeridian is refused as malformed. The README's rules section
  says what to do about both; geodesic distance belongs with the
  geometry, in geo-nv.
- **No R*-tree.** Guttman's insertion is what is specified here.
  `margin` and `enlargement` are both published, which is most of what
  the variant needs, and adding it changes no signature.
- **Removal does not rebalance**, and neither does insertion. A tree
  that has been heavily updated is rebuilt from `entries` or `items`.
