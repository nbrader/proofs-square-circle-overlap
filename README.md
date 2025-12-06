# proofs-square-circle-overlap

Formal verification in Rocq (formerly Coq) of the overlap area between a circle
and a square, plus a general interactive visualization for arbitrary polygon-circle
overlaps. The project provides case-by-case analysis for squares, a general
boundary-walk algorithm for polygons, and reference implementations in Haskell
and Python.

## Problem statement

**Core verification target:** Given a circle of radius R and a square with side
length 2s (both centered at the origin), compute the area of their intersection.
The solution requires careful geometric decomposition based on the relationship
between R and s.

**General visualization:** The interactive viewer extends this to arbitrary convex
and non-convex polygons, computing overlap areas using a boundary-walk algorithm
that works for any polygon shape and circle position.

## Interactive visualization: `circle-square-overlap.html`

A fully interactive web application that visualizes polygon-circle overlap in both
2D and 3D. Simply open the file in a modern web browser—no build step required.

**Features:**
- **Multiple shapes:** Square, rectangle, triangle, pentagon, hexagon, octagon,
  custom polygons (draw your own or define via JSON)
- **2D interactive view:** Drag the circle center, zoom/pan with mouse wheel,
  see real-time overlap area calculation with visual decomposition into triangles
  and sectors
- **3D surface plot:** Visualize the overlap kernel A(cx, cy) as a surface over
  all possible circle center positions (red/blue axes = inputs, green axis = overlap area)
- **Animated overlap tracing:** Step-by-step visualization of the boundary-walk
  algorithm showing how triangles and sectors are computed
- **Exact geometric tests:** Uses precise point-to-segment distance calculations
  (not sampling) for numerical stability—eliminates flickering spikes when
  adjusting parameters

The viewer computes A(cx, cy) = ∫ 1_P(u) · 1_C(u−(cx,cy)) du — the convolution
of a polygon indicator function 1_P and circle indicator 1_C at offset (cx, cy).
Both the 2D drag view and 3D surface plot evaluate this kernel for different
circle centers.

### Boundary-walk algorithm

The general algorithm (implemented in the HTML viewer):
1. **Find intersections:** Locate all circle–polygon edge intersection points
2. **Edge triangles:** Walk polygon edges; for each segment inside the circle,
   add the signed triangle area from circle center to segment endpoints
3. **Arc sectors:** Walk circle arcs between consecutive intersections; if the
   arc midpoint lies inside the polygon, add the circular sector area
4. **Special cases** with exact geometric tests:
   - **Polygon fully inside circle:** All vertices closer than radius → return polygon area
   - **Circle fully inside polygon:** Center inside AND all edges farther than radius → return πR²
   - **No overlap:** Center outside AND all vertices/edges farther than radius → return 0

This structure generalizes to any polygon: the algorithm alternates between
straight-line segments (triangles) and circular arcs (sectors), using signed
areas to correctly handle complex boundary configurations.

### Mathematical foundation

The algorithm is based on **Green's theorem**, which relates area integrals to
boundary line integrals:

∬_R dA = ∮_∂R x dy = -∮_∂R y dx = (1/2) ∮_∂R (x dy - y dx)

For a simple polygon with vertices (x₀,y₀), ..., (xₙ₋₁,yₙ₋₁), this yields the
**shoelace formula**:

A = (1/2) Σᵢ (xᵢ · yᵢ₊₁ - xᵢ₊₁ · yᵢ)

Our algorithm extends this to **mixed boundaries** containing both straight edges
and circular arcs:

1. **Line segments:** Each edge segment inside the circle contributes a signed
   triangular area from the circle center to the segment endpoints (equivalent to
   the cross product term in the shoelace formula)

2. **Circular arcs:** Each arc segment inside the polygon contributes a circular
   sector area (the generalization of the line integral to curved boundaries)

This is effectively a **generalized shoelace formula for circle-polygon overlap**
using signed sector-triangle decomposition. The signed area formulation
automatically handles orientation and ensures correctness for both convex and
non-convex polygons.

## Goal: integrating over all circle centers

If the objective is to integrate the overlap area over every possible circle
center z in the plane, define A(z) = ∫ 1_S(u) · 1_C(u−z) du (a convolution).
Swapping the integrals yields

∫ A(z) dz = ∫ 1_S(u) (∫ 1_C(u−z) dz) du = (area of C) · (area of S)
          = (πR²) · (4s²).

Thus, over all of ℝ² the total integral is just the product of the two areas,
so the eight positional cases collapse. If circle centers are restricted to a
bounded window instead, integrate this same convolution only over that window.

## Case decomposition (square-specific)

The image `Rough/Area of overlap between circle and square - 2D (Tidy Working).png`
illustrates the eight distinct cases that arise for the square-circle case:

1. **Full circle contained** (s >> R): Overlap area = πR²
2. **Circle extends beyond top/bottom** (moderate s):
   Area = πR² - sector(y) + triangle(y)
3. **Circle extends beyond top and right** (smaller s):
   Area = πR² - sector(x) + triangle1(x) - sector2(y) + triangle2(y)
4. **Complex asymmetric case** (s ≈ R):
   Uses double integral formula involving arcsin and algebraic terms
5. **Circle extends beyond bottom** (different moderate s):
   Area = sector(y) - triangle(y)
6. **Circle mostly outside, lower configuration**:
   Area = πR² - LowerRightAreaLikeD(x,y) - LeftSegment(x)
7. **Circle mostly outside, upper-right configuration**:
   Area = UpperRightSector(x,y) - UpperTriangle(x,y) - RightTriangle(x,y)
8. **No overlap** (s << R or circle too small): Overlap area = 0

Each case employs a combination of circular sectors, triangular corrections,
and in the most general scenario a double integral over the overlapping domain.
The general polygon algorithm unifies these cases through the boundary-walk approach.

## Repository layout

- **`circle-square-overlap.html`** – interactive web application for visualizing
  polygon-circle overlap with 2D/3D views, animation, and multiple shape support
- **`Spec.txt`** – specification of the overlap algorithm, correctness invariants,
  and documented implementation notes
- **`Rocq/`** – the formal development
  - `SquareCircleOverlap.v`: definitions of geometric primitives (sectors,
    triangles, segment areas), the case analysis, and the main area formula
  - `_CoqProject`: logical load paths (`-Q . SquareCircleOverlap`) and file list
- **`Haskell/`** – informal reference implementation that mirrors the case analysis
  for numerical experimentation in GHCi
- **`Python/`** – quick-running sanity checks that empirically validate the formulas
  against numerical integration
- **`Rough/`** – working diagrams and visual aids, including the comprehensive
  8-case illustration

## Building the Rocq proofs

```bash
cd Rocq
coq_makefile -f _CoqProject -o Makefile
make                    # compiles all .v files
make clean              # remove artifacts
```

The project targets Rocq 8.20+ (Coq 8.20+). All files live under the
`SquareCircleOverlap` namespace as configured in `_CoqProject`.

## Haskell quick start

```bash
cd Haskell
ghci SquareCircleOverlap.hs
Prelude> overlapArea 1.0 1.5   -- circle radius 1.0, square half-side 1.5
```

This implementation is not formally verified but is useful for exploring the
geometric formulas and generating visualizations.

## Python sanity checks

The scripts in `Python/` provide numerical integration baselines:

```bash
cd Python
python3 test_overlap_cases.py
python3 validate_formulas.py
```

They compare the analytic formulas against Monte Carlo and quadrature methods
to ensure correctness before formalizing in Rocq.

## Current status

- **Interactive visualization (`circle-square-overlap.html`):** Fully functional
  with support for arbitrary polygons, exact geometric tests, and smooth numerical
  behavior across all parameter ranges
- **Rocq formal proofs (`Rocq/SquareCircleOverlap.v`):** Under active development,
  focusing on the 8-case square-circle decomposition
- **Haskell and Python implementations:** Match the square-specific case analysis
  and serve as numerical references
- **General polygon algorithm:** Implemented and tested in the HTML viewer;
  boundary-walk approach confirmed to handle convex and non-convex shapes correctly

To work on the development interactively, use your preferred Rocq/Coq IDE
(CoqIDE, VS Code + VsCoq, Proof General), reload
`Rocq/SquareCircleOverlap.v`, and re-run `make` after edits to ensure the
project compiles.

## Recent improvements

- Replaced unreliable 8-point circle sampling with exact geometric distance tests
  to eliminate numerical spikes and ensure stable behavior across all radius values
- Implemented precise point-to-segment distance calculations for special case detection
- Added comprehensive specification documentation in `Spec.txt`
