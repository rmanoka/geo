---
mathjax: true
blog: true
---

# Line Crossings

Identifying all intersections of a collection of
line-segments is an important sub-routine in many comp.
geom. algorithms. The simplest implementation of this ---
checking all pairs of segments --- takes $O(n^2)$ time. This
problem has been studied extensively, and faster algorithms
are known.

# [Bentley-Ottman] Algorithm

The Bentley-Ottman algorithm is a classical algorithm that
lists all intersections in time $O((n+k) \log(n))$, where
$k$ is the number of intersecting (or overlapping) pairs.
This is faster than checking all-pairs for the typical case:
$k \ll O(n^2).$ It is also a practical algorithm, and
appears in some variation in many geom. libraries.

The algorithm sweeps a vertical (infinite) line from left
($x = -\infty$) to right ($x = \infty$), while keeping track
of segments that intersect the line at any specific instant
ordered by the $y$ coordinate of its intersection with the
sweep line. The key observations are that:

1. any two intersecting segments were next to each other
   just before the sweep line touches the intersecting point.
1. the set of segments that intersect the sweep line, and
   their relative ordering change only at discrete points:
   either the end points of the segments, or their
   intersections.

The first observation allows us to enumerate all
intersections of a segment by only checking the previous and
next segment along the sweep line. The second observation
allows us to efficiently make this check only when the
ordering changes: when the sweep reaches the start, end or
an intersection of a segment. As the intersections are not
known before hand, the we simulate the sweep using a
min-heap data-structure initialized with the end points, and
add intersections are as they are discovered.

When the left end of a segment hits the sweep-line, we check
for intersection with the segments above and below it along
the sweep-line. Similarily, at the right end, we check for
intersection between the segments above and below it. At
each intersection, the ordering of the lines is adjusted,
and we look for intersection between the segment now on top,
and the segment above it; as well as between the bottom one
and the segment below it.

## Primer: Simple Case

The details of the data-structures, and the algorithm are
easier to explain under the following simplifying
assumption. We will see how to lift these assumptions later.

1. Only identify intersections in the interior of segments.
1. No three input segments intersect at the same point.
1. All computations of intersections are exact

As mentioned above, we use a min-heap to simulate the sweep.
The ordering of the points is by the x coordinate, and if
equal, by the y coordinate. The ordering by y coordinates is
important to handle vertical segments (eg. two overlapping
vertical segments). Henceforth, we denote a segment as $(p,
q)$ where $p$ and $q$ are the _left_ and _right_ end points;
that is, $p < q$ as per the above ordering. If $p = q$, it
does not have an interior (but we handle point-line
intersection later).

### Ordering the Segments

To query segments above and below a given segment along the
sweep-line, we track the set of segments intersecting the
current sweep line (aka _active segments_) as an ordered
set. However, the ordering of the segments is not fixed, and
changes with the position of the sweep line. This violates
the requirements of typical data-structures used to
represent an ordered set such as `BTreeSet`.

Instead, we design a partial ordering, and show conditions
under which it is a total ordering. Then, we ensure that the
set of active segments always satisfy these conditions. The
partial ordering is defined for two segments $(p_1, q_1),$
and $(p_2, q_2)$ only if both both $p_1, p_2$ are strictly
smaller than $q_1, q_2$. It is easier to express when $p_1
\le p_2$, and the other case is handle by reversing.

```rust
fn cmp_simple(s1: Segment, s2: Segment) -> Ordering {
    let (p1, q1) = s1;
    let (p2, q2) = s2;
    // pre-condition: p1 <= p2
    // pre-condition: p2 < q1

    fn orientation_as_ordering(o: Orientation) -> Ordering {
        match o {
            CounterClockwise => Less,
            Clockwise => Greater,
            Collinear => Equal,
        }
    }

    orientation_as_ordering(orient2d(p1, q1, p2))
        .then_with(|| {
            // p1, q1, p2 are collinear
            // use q2 to define the ordering
            orientation_as_ordering(orient2d(p1, q1, q2)
        })
}
```

The assumption $p_i < q_j$ for $i, j \in \\{1, 2\\}$ implies
that there is a sweep line passing through both segments.
The above logic orders the two segments along any common
sweep-line that occurs before the intersection of the two
segments. Thus, for any set of segments satisfying the above
condition, the ordering is in fact a _total ordering_ and
compatible to any sweep-line before any intersections among
the segments.

### Tracking Active Segments

To track the active segments, we insert a segment into the
set when we encounter the left end point, and remove it at
the right end point. The lexicographic ordering of points
described above ensures that the active segments satisfy:
$p_i \le q_j$ for any pair $i, j$ of active segments. To
make the inequality strict, we must also ensure that the
heap orders any right end-point $q_j$ before all coinciding
left end-points $p_i = q_j.$

As described in the outline, we look for intersections
before inserting a new segment, or after removing a segment.
Since interior intersections are not allowed among the
active segments, we break the segment with interior
intersection.  There are two cases to consider here.

1. Intersection at a point: in this case, we break the
   segments into exactly 4 segments as the point is in the
   interior. At most two of these segments are already
   active whose end point must be shortened to the
   intersection point.
1. Overlapping segments: in this case we break it into one
   overlapping segment, and 2 additional segments. One of
   these is already active and must be shortened.

In either case, the current point $x$ is strictly smaller
than the intersection or the first overlap point. This is
because the intersections are in the interior of the
segments. Thus, any existing existing segments can be
shortened to the intersections without violating the
constraint on the end points. It also does not affect the
ordering of the active segments (TODO: proof).

### Correctness

Prove that each intersection is visited, and exactly once.

## Handling Degeneracy

When breaking segments, mark segments that are
continuations. Record intersections by collecting all heap
pops at the same point.

TODO: Work out handling of overlapping segments.

TODO: Explain additional tie-breaks in ordering for the
Martinez-Rueda and related algorithms.

## Handling Inexact Constructions

For inexact computations (eg. `f64`), we must ensure that
when we compute the intersection, the smaller segments are
still ordered consistently. This is not possible in general
though. We could detect inconsistencies and error out
suggesting that higher precision data-type is to be used.


[Bentley-Ottman]: //en.wikipedia.org/wiki/Bentley%E2%80%93Ottmann_algorithm
