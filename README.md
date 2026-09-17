# spatial-nv

A spatial index answers questions about position: what is inside this
area, what is nearest to this point, what are the closest five. It does
it by storing the items in a tree whose shape follows the geometry, so
a query touches a few dozen of them instead of all of them. This
package brings two of the standard ones to novo-lang — the **R-tree**,
described by Antonin Guttman in 1984, and the **k-d tree**, described
by Jon Bentley in 1975. They are the two that
[rstar](https://docs.rs/rstar) and the
[rtree](https://en.wikipedia.org/wiki/R-tree) family are built on.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What it is

A **bounding box** is the smallest axis-aligned box that holds a
shape: two numbers per axis, a lower bound and an upper bound. It is
what every spatial index stores instead of the shape itself, because
the two questions a search has to ask about a whole subtree — could
anything in here overlap that, and how close could anything in here be
— are cheap on a box and expensive on a road, a polygon or a
photograph's footprint.

An **R-tree** is a balanced tree of boxes. Each leaf holds up to a
fixed number of items, each branch holds up to the same number of
children, and every node carries the box that holds everything beneath
it. A search descends only into the children whose boxes could contain
an answer. Sibling boxes may overlap, which is what lets the tree index
extents rather than points, and which is also why building it well
matters: a tree whose sibling boxes overlap a lot makes a search open
two branches where one would do.

A **k-d tree** is a binary tree over points. Each node holds one point
and splits the space on one **axis** — everything with a smaller
coordinate on that axis goes left, everything larger goes right — and
the axis changes with the depth, cycling through the coordinates. Its
subtrees never overlap, so a search opens strictly less of the tree
than an R-tree's would; the price is that it indexes positions only.

**Bulk loading** builds an index from a set that is already known, in
one pass. The method here is **Sort-Tile-Recursive** packing: sort the
items by one coordinate, cut them into slices, sort each slice by the
next coordinate, and fill the leaves from the result. Every leaf comes
out full and the boxes overlap much less than the same items inserted
one at a time would produce. **Insertion** adds one item to an index
that already exists, which is what a growing set needs and what
produces the worse-packed tree.

A **handle** is an integer: the position of an item in the caller's own
list. This package stores handles and never the items.

## Install

```
novo pkg add spatial-nv
```

## Example

```novo
use spatialbox
use spatialrtree
use spatialquery

fn main() [io]
    // The caller's own data. The index never sees these strings.
    let names = ["depot", "market", "harbour"]

    // One box per item, keyed by that item's position in `names`.
    match spatialbox.box_of([0.0, 0.0], [1.0, 1.0])
        Err(e) => println(e.message())
        Ok(first) =>
            match spatialbox.box_of([4.0, 0.0], [5.0, 1.0])
                Err(e) => println(e.message())
                Ok(second) =>
                    // Build the index in one pass over the known set.
                    match spatialrtree.bulk_load([spatialrtree.entry(0, first),
                                                  spatialrtree.entry(1, second)], 8)
                        Err(e) => println(e.message())
                        Ok(index) =>
                            match spatialbox.point([6.0, 0.5])
                                Err(e) => println(e.message())
                                Ok(here) =>
                                    // Ask what is closest. The answer is a
                                    // handle and the distance already computed.
                                    match spatialquery.rtree_nearest(index, here)
                                        Err(e)      => println(e.message())
                                        Ok(None)    => println("nothing indexed")
                                        Ok(Some(h)) => println("${names[h.handle]} at ${h.distance}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: spatial-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `spatialfault` | Every way a call is refused, and the numbers that refused it. |
| `spatialbox` | A point, a box, and the algebra both trees are built out of: union, intersection, overlap, containment, area, margin, enlargement, centre and distance. |
| `spatialrtree` | The R-tree: its node, its two constructors, insertion, removal and the facts about a built tree. |
| `spatialkdtree` | The k-d tree: its node, the median build, insertion, and the axis rule. |
| `spatialquery` | The four queries — range, radius, nearest and k-nearest — asked of either tree. |

## How to choose an entry point

**Indexing extents — roads, parcels, rectangles on a screen, anything
that occupies an area: `spatialrtree`.** A k-d tree cannot represent an
item that is not a point.

**Indexing positions — sensors, cities, feature vectors:
`spatialkdtree`.** Where both would work, this is the better index. Its
nodes carry no boxes, so it is smaller, and its subtrees do not
overlap, so a search opens less of it.

**The set is known: `spatialrtree.bulk_load` or `spatialkdtree.build`.**
Both build in one pass and both produce a better-balanced tree than
insertion does.

**The set grows: `spatialrtree.insert` or `spatialkdtree.insert`.**
Both answer a new index rather than changing the old one. Neither
rebalances, so a tree that has been inserted into many times is worth
rebuilding — `entries` or `items` gives everything back, and the bulk
constructor takes it.

**A box question is `rtree_range`; a distance question is
`rtree_radius`.** They are different shapes and they give different
answers.

## The rules a user needs

1. **The index stores handles, and keeping them valid is yours.** A
   handle is a position in your list. If you reorder, insert into or
   delete from that list, every handle in the index means something
   else, and the index is stale. Rebuild it, or use handles that are
   stable identifiers into a map you control rather than positions in
   a list you mutate.
2. **Every distance in this package is planar.** Coordinates are
   treated as points in a flat space and distance is Euclidean.
   Nothing here knows about the curvature of anything.
3. **Longitude and latitude are welcome and are not metres.** A degree
   of longitude is about 111 km at the equator and about 55 km at 60°
   north, so a nearest-neighbour result in degrees is not a
   nearest-neighbour result in distance away from the equator. Project
   the coordinates first when the ranking has to be by true distance,
   or use the index as a coarse filter and re-rank the few hits it
   returns with a geodesic distance.
4. **A box that crosses the antimeridian has a lower bound above its
   upper bound**, and `spatialbox.box_of` refuses it as malformed. Cut
   such a box into the two boxes either side of 180° and query with
   both, which is the same thing RFC 7946 requires of a geometry that
   crosses it.
5. **Touching counts.** Two boxes that meet exactly at a face overlap,
   and a point on a box's boundary is inside it. Every query boundary
   follows that convention.
6. **An empty answer is a success.** A query on an empty index, an area
   that holds nothing, and a k-nearest query asking for more than the
   index holds all succeed — with `None`, an empty list, and however
   many there are. None of them is an error.
7. **A hit carries its distance.** The searches computed it to rank and
   to prune; `SpatialHit` hands it over so you do not compute it again
   over items this package does not hold. The range query is the
   exception and answers bare handles, because "inside this box" has no
   distance.
8. **Insertion does not rebalance.** Points inserted into a k-d tree in
   sorted order build a tree that is a linked list. Read `depth`
   against `size` to notice, and rebuild.
9. **Node capacity is a trade, and 8 is the default.** A larger
   capacity makes a shallower tree and more work at each node. It must
   be at least 2; 1 is refused, because a node holding one entry makes
   a tree that is a linked list.

## What is not included

- **The caller's items.** By design, and rule 1 above is the
  consequence. A generic `RtreeIndex<T>` holding the items is what a
  language with working generics would offer; ndarray-nv measured this
  toolchain and published the result, which is that a generic container
  crossing a module boundary stops the compiler or fails IR
  verification — and a library is nothing but a module boundary. The
  handle is not only the available design, it is a good one: an item
  can be in three indexes without being stored three times.
- **Geometry.** No polygons, no intersection of shapes, no area of
  anything but a box, no coordinate reference systems and no
  projections. `geo-nv` is that package, and this one takes its
  bounding boxes as plain numbers rather than depending on it — an
  index over longitudes and an index over pixel coordinates are the
  same data structure.
- **Geodesic distance.** See rule 3. Distance on an ellipsoid belongs
  with the geometry, and re-ranking an index's hits with it is the
  intended way to combine the two.
- **R*-trees.** The R*-tree's reinsertion and its margin-based split
  produce measurably better trees than Guttman's original, at the cost
  of a more complex insertion. `spatialbox.margin` and
  `spatialbox.enlargement` are both published, which is most of what
  the variant needs, and adding it later changes no signature here.
- **Persistence.** Nothing here reads or writes a file. An index is a
  value; a caller that wants one on disk serialises the entries and
  bulk-loads them back, which is cheaper than storing the tree.
- **Concurrent or incremental bulk loading**, and the deletion
  rebalancing that would keep a heavily-deleted tree packed.

## Related packages

- **`geo-nv`** holds the geometries — points, lines, polygons, GeoJSON
  and WKT — and the geodesic distances. This package is the index those
  geometries are looked up through; the dependency runs from a caller
  that has both, not between them.
- **`cluster-nv`** groups points that are near each other. A spatial
  index is what makes DBSCAN's neighbourhood query fast, so the two
  compose, but neither depends on the other.
- **`ndarray-nv`** is the package for coordinates by the million in a
  matrix. This one is for a handful of numbers per item, compared
  rather than multiplied.

## Tests

The suite is written against the signatures and is red by
construction: every assertion reaches a `todo()`. The fixtures are
built so that every expected number is exact and no assertion needs a
tolerance: five unit boxes in a row two units apart, so each distance
is a whole number; the 3-4-5 right triangle, so a planar distance is
exactly 5; and seven points spread 1 to 7 on one axis, whose median
split is determined at every level, so the k-d tree's root, its depth
and its balance can all be asserted rather than sampled. The empty
index, the zero-width box, the zero radius and the k larger than the
index are each asserted to succeed.

## Implementation status

| Module | Declared | Implemented |
| --- | --- | --- |
| `spatialfault` | 8 variants, `message` | no |
| `spatialbox` | 17 functions | no |
| `spatialrtree` | 13 functions | no |
| `spatialkdtree` | 10 functions | no |
| `spatialquery` | 10 functions | no |

## Licence

Apache-2.0. See [LICENSE](LICENSE).
