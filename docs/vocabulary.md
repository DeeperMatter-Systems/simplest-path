# Shape and constraint vocabulary

**Status: proposed design notes for review. Documentation only.**

This file records the shape and constraint vocabulary for Simplest Path and corrects the earlier vocabulary page where that page was technically wrong. Nothing here is an implemented schema, checker, solver, prompt, or file format. Phase placements are proposed scope, not a benchmark result. Numbers labeled as defaults are tunable illustrations, not validated thresholds. No accuracy, success rate, or dataset result is claimed.

## Scope

The intended pictures are logos and simple technical sketches.

**Input.** A raster PNG, or a rough traced SVG.

**Interfaces.** A sketch-input interface and a traced-SVG cleanup interface. Both emit the same constrained model.

**Output.** Clean lines, circles, and circular arcs, plus an editable constraint set and SVG.

**Division of work.**

1. A vision step proposes primitives and relationships.
2. Deterministic numeric checks score geometric plausibility.
3. A person accepts, rejects, or edits those candidates.
4. A solver enforces only the accepted relationships.

A small residual means the drawing is geometrically close to a relationship. It does not mean the designer intended that relationship. Nearly horizontal strokes, nearly equal radii, and nearly tangent curves stay candidates until review accepts them.

## Minimal representation

The goal is a small set of primitives and accepted constraints that preserves three things:

- **Shape.** The cleaned curves still follow the sketch.
- **Topology.** Endpoints that meet still meet; a stroke that stops stays stopped.
- **Design intent.** Parallelism, tangency, equal size, symmetry, and driving dimensions that a reviewer actually accepts.

That is a different goal from removing every degree of freedom. A logo may still translate, rotate, or, when no driving size was accepted, scale. FreeCAD's sketcher documentation describes a fully constrained sketch as one with no freedom left. That CAD aim is cited in [References](#references) so it is not confused with this goal.

Extra constraints that are only numerically plausible should not be added to eat degrees of freedom. Redundant constraints that restate one accepted relationship can make a solver fail. One accepted encoding is enough.

A future solver still needs a gauge anchor so the sketch does not drift under rigid motion. That anchor is an implementation concern. It is not a user-facing fix or lock, and user-facing fix/lock stays deferred.

## Pipeline

```text
PNG or rough SVG
    -> proposed primitives and relationships
    -> numeric plausibility residuals
    -> human review (accept, reject, edit targets)
    -> solver on the accepted set only
    -> lines, circles, arcs, editable constraints, SVG
```

Ellipse, elliptical arc, Bézier, and B-spline input is either kept as preserved unsupported geometry or rejected explicitly. It is not replaced by lines and arcs in secret.

Solver status and the displacement from the input are both monitored. Redundant, inconsistent, or non-converged solves revert to the pre-solve geometry and return the set to review. A solve that converges by moving the sketch far from the input also reverts. No displacement budget is specified; choosing one is future work.

## Primitives

Coordinates are written in one declared frame before any residual is computed. The illustrative JSON below declares pixels, x to the right, and y up. A raster file whose pixel axis points down has to be flipped into that frame during normalization. The flip itself is future work; the frame has to be declared so angles have a meaning.

### Version 1

| Entity | Parameters | Meaning |
| --- | --- | --- |
| Point | `x`, `y` | Endpoint, center, or explicit anchor |
| Line segment | `start`, `end` | Finite segment. Direction is `end − start`. Length is the distance between the endpoints |
| Circle | `center`, `radius` | Full circle |
| Circular arc | `center`, `radius`, `start_deg`, `sweep_deg` | The circular curve from the start angle through a signed sweep. A positive sweep is counterclockwise in the declared frame. This is an unfilled curve, not a filled sector |

A zero sweep is degenerate and is rejected. A full turn is a circle, not an arc. Whether a stored sweep of exactly ±360° should be rewritten as a circle is an open decision; the proposal is to keep full turns as circles.

SolveSpace specifies an arc by center, two endpoints, and a normal, drawn counterclockwise from the first endpoint to the second ([SolveSpace reference](https://solvespace.com/ref.pl)). The parameters above can describe that same curve. The earlier vocabulary page called an arc "the same as a circle sector." That description is withdrawn: a sector is a filled region, and this entity is the curve.

### Later, and not approximated

| Entity | Proposed handling |
| --- | --- |
| Ellipse, elliptical arc | Unsupported. Preserve the input or mark it unsupported |
| Bézier, B-spline | Unsupported. Preserve the input or mark it unsupported |

### Helpers

Rectangles and polygons are construction helpers over line segments and ordinary constraints. They are not version-1 primitives. SolveSpace's rectangle command is the same kind of helper: four line segments that could have been drawn and constrained by hand.

## Proposed priority

Phases are a proposed cut of scope. They are not a measured ranking and not a conclusion from a benchmark.

Two scoping judgments sit on top of that cut:

- Horizontal, vertical, and concentric constraints are inexpensive and high value for axis-aligned technical sketches.
- Symmetry about an explicit axis is high value for logos.

FreeCAD's sketcher best-practices text says horizontal and vertical distance constraints are computationally cheaper than the general length constraint, and it lists horizontal and vertical constraints among the ones to prefer. That is FreeCAD's guidance for its own sketcher, not a measurement of this project. It is consistent with the scoping judgment above, and the phase column remains a proposal.

| Relation | Entities | Proposed phase | Scoping note |
| --- | --- | --- | --- |
| `coincident` | point, point | 1 | Shared corners and merged nodes. Subreferences name which point: `start`, `end`, or `center` |
| `point_on` | point and a line, circle, or arc | 1 | Endpoint on an edge, or a center on an axis. `extent` says whether the line test is the finite segment or the supporting line |
| `parallel` | line, line | 1 | Direction match, including opposite directions |
| `perpendicular` | line, line | 1 | Right angles in marks and frames |
| `tangent` | line–circle, line–arc, circle–circle, arc–arc | 1 | Finite contact required. Circle pairs store an internal or external branch |
| `equal_length` | line, line | 1 | Repeated stroke lengths. Length is the endpoint distance |
| `equal_radius` | circle or arc, circle or arc | 1 | Matched rounds |
| `distance` | point–point, point–line, or parallel line–line | 1 | Driving dimension. Target and units are required. Two lines have a single separation only when they are parallel |
| `radius`, `diameter` | circle or arc | 1 | Driving size. Target and units are required |
| `angle` | two directed lines | 1 | Driving angle. The orientation convention is stored with the constraint |
| `concentric` | circle or arc, circle or arc | 2 | Shared centers. Inexpensive for technical sketches. Equivalent to coincident centers when centers are point subreferences, and still not a tangency |
| `collinear` | two lines, or three or more points | 2 | Broken strokes brought onto one supporting line. Overlap is not required |
| `midpoint` | point, line | 2 | The point is the average of the segment endpoints |
| `symmetry` | corresponding points, lines, circles, or arcs, plus an explicit axis | 2 | High value for logos. The axis is a referenced line, not an inferred page axis |
| `equal_angle` | two pairs of directed lines | 2 | The same orientation convention as `angle` |
| `equal_sweep` | two arcs | 2 | Compares signed sweeps |
| `offset` | two lines, or two circles or arcs | 2 | Parallel lines, or concentric circles, at a target distance. Side or branch is stored |
| `horizontal` | line, or a pair of points, in the page frame | 2, optional | Inexpensive for technical sketches. A free-rotated logo can sit near the page axis without this being design intent |
| `vertical` | line, or a pair of points, in the page frame | 2, optional | Same as horizontal, against the page y axis |
| `length_ratio` | two lines | defer | Present in SolveSpace. Rare for logos and simple technical sketches |
| patterns | linear or circular repeats | defer | CAD-heavy relative to this scope |
| 3D, workplanes | — | defer | Out of scope. SolveSpace's projected constraints and workplanes are not part of this model |
| `fix` / `lock`, user-facing | point or entity | defer | A reviewer-facing anchor. Distinct from the internal gauge anchor a solver needs |

`length` is not a separate relation. It is `distance` between a segment's endpoints. The earlier page listed it as its own row.

## Numeric verification

Each residual is a candidate check. Under the proposed rule, a candidate is plausible when the residual is within the tolerance for that quantity. Plausible is not accepted. Degenerate inputs are rejected before the residual is treated as evidence.

The 2D cross product of directions `u` and `v` is `ux vy − uy vx`. The dot product is `ux vx + uy vy`.

| Relation | Residual | Extra check |
| --- | --- | --- |
| `coincident` | distance between the two points | The points come from `start`, `end`, `center`, or a point entity |
| `point_on` a line | perpendicular distance from the point to the supporting line | `extent: "finite"` also requires the foot to lie on the segment, with along-segment slack of the distance tolerance. A zero-length segment is rejected. `extent: "infinite"` skips the range check |
| `point_on` a circle | absolute difference between the center distance and the radius | Non-positive radius is rejected |
| `point_on` an arc | the circle residual, and the point's angle must lie in the signed sweep | A point on the supporting circle and outside the sweep fails. Endpoints are included. Angular slack at the ends uses the angle tolerance |
| `parallel` | absolute value of the normalized cross product, which is `\|sin θ\|` | Zero-length directions are rejected. Opposite directions are parallel. A signed angle near 180° is not a failure |
| `perpendicular` | absolute value of the normalized dot product, which is `\|cos θ\|` | Zero-length directions are rejected. Either right angle passes |
| `equal_length` | absolute difference of the two endpoint distances | |
| `equal_radius` | absolute difference of the two radii | |
| `concentric` | distance between centers | This is not a tangency test |
| `collinear` lines | the parallel residual, and the perpendicular distance between the supporting lines | Separated segments on one line still pass. Zero-length directions are rejected |
| `collinear` points | perpendicular distance of each other point to the line through an extreme pair | The extreme pair must have non-zero length |
| `midpoint` | distance from the point to `(a + b) / 2`, where `a` and `b` are the segment endpoints | Equal distance to `a` and `b` is the perpendicular bisector. That test is the wrong one, and it is not used |
| `symmetry` | after reflection across the axis: distance between corresponding endpoints and centers; absolute radius difference; absolute difference between the reflected signed sweep and the other sweep | The axis is the supporting line of an explicit segment. Reflection reverses sweep sign. Endpoint correspondence is stored. Assuming `start` maps to `start` is valid only when both parameterizations run in mirrored order |
| `distance` | absolute difference between the measured distance and the target | Point–point is Euclidean. Point–line is the perpendicular distance, using the same finite-segment rule as `point_on` when `extent` is finite. Line–line separation is defined only for parallel lines; other line pairs are rejected. Units are required |
| `radius` | absolute difference between the radius and the target | Units are required |
| `diameter` | absolute difference between `2r` and the target | Units are required |
| `angle` | absolute wrapped difference between the measured oriented angle and the target | Convention below. A supplementary target is a different constraint |
| `equal_angle` | absolute wrapped difference of the two oriented angles | Four directed lines, two pairs, same convention as `angle` |
| `equal_sweep` | absolute difference of the signed sweeps | A mirrored pair has opposite signs and fails this test. Symmetry is the relation that flips orientation |
| `offset`, lines | the parallel residual, and the absolute difference between the perpendicular separation and the target | Side is stored. Units are required |
| `offset`, circles | the concentric residual, and the absolute difference between the absolute radius difference and the target | How an offset arc inherits sweep is an open decision |
| `horizontal` | angular residual of the segment, or of the segment joining two points, against the page x axis, ignoring direction reversal | Zero length is rejected. The page frame must be the declared frame |
| `vertical` | the same residual against the page y axis | Same |
| `tangent`, line and circle or arc | absolute difference between the center-to-line perpendicular distance and the radius | The foot of the perpendicular is the contact. It must lie on the finite segment. For an arc, the contact angle must lie in the signed sweep. Side of the directed line is stored. A shared-endpoint join also requires the radius at that endpoint to be perpendicular to the line |
| `tangent`, two circles or arcs | external: absolute difference between the center distance and `r1 + r2`. Internal: absolute difference between the center distance and `\|r1 − r2\|` | The branch is stored. The contact must lie on both finite curves, which for arcs means both sweep tests. Coincident centers are rejected as tangent candidates |

### Angle convention

Proposed convention, not a locked format:

- Line direction is `end − start`.
- The oriented angle from direction `u` to direction `v` is `atan2(cross, dot)`, in `(−180°, 180°]`.
- The residual is the absolute value of that measured angle minus the target, wrapped back into `(−180°, 180°]`.
- Reversing a segment flips the sign of a directed angle. Endpoint order is part of the shape.
- `parallel` and `perpendicular` use the absolute sine and cosine tests so they stay insensitive to reversal. The general `angle` constraint does not.

SolveSpace stores an angle between 0° and 180° and has a separate command for the other supplementary angle, because a new angle constraint otherwise chooses the supplementary angle arbitrarily. The convention above stores the intended signed angle instead of repairing that choice later. It is a proposal for this model, not a description of SolveSpace's solver.

### Tangency details

Radius sum, radius difference, or a single line-to-center distance does not finish the test.

- On bounded geometry the contact has to lie on the finite segment and, for an arc, inside the signed sweep. An infinite line or a full circle can be tangent while the actual stroke misses the contact.
- On a shared-endpoint join, the curves already meet because of coincidence. Tangency is the extra direction match at that point. A radius-sum test can pass at a different contact, or fail to express smoothness at the join.
- Internal and external circle tangency are different branches and both are stored. External contact lies between the centers, at distance `r1 + r2`, with the radius vectors anti-parallel. Internal contact lies at distance `\|r1 − r2\|`, with the radius vectors parallel.
- Coincident centers are not ordinary tangent circles. Internal tangency of equal radii collapses to coincident centers. That candidate is concentric plus `equal_radius`, and it is not emitted as `tangent`.
- Equal radii with centers `r1 + r2` apart are ordinary external tangency. Equal radii alone do not decide.

SolveSpace's parallel/tangent constraint documents a narrower case: the curves already share an endpoint, and the constraint then forces tangency there. Interior contact on a finite segment is in scope here, so that precondition is not the definition.

FreeCAD's tangent tool makes two selected lines collinear. This vocabulary keeps `tangent` and `collinear` as different relations.

### Worked residuals from the illustrative JSON

The sample below is constructed so the following are exact in integer arithmetic. This is a consistency check of the example, not a dataset.

- `L1` and `L2` both have direction `(120, 0)`. The normalized cross product is 0, so the parallel residual is 0.
- `P1` is `(100, 60)`. The midpoint `(start(L1) + end(L1)) / 2` is `(100, 60)`.
- `C1` and `C2` have centers 80 apart and radii 40 and 40. The external tangent residual is `|80 − 80| = 0`. The contact `(140, 100)` lies on both circles. The centers are not coincident.
- `A1` sweeps −90° and `A2` sweeps +90°. The equal-sweep residual is 180°. That candidate is rejected. Reflection of `A1` across `S1` (`x = 100`) maps its center, endpoints, radius, and reversed sweep onto `A2`.
- The point `(140, 100)` lies on `C1` and on `A2`. Its angle from `A1`'s center is 0°, which is outside `A1`'s sweep from 180° through −90°. A `point_on` test against `A1` fails on sweep membership while the radial residual is 0.

## Tolerances

Proposed policy:

- Normalize every coordinate into the declared frame before comparing numbers. Pixel coordinates and SVG user units are not mixed raw.
- Combine an absolute floor (pixel quantization, stroke thickness, trace jitter) with a term scaled by local feature size: segment length, radius, or the local spacing of the pair. The page diagonal is the wrong scale. It swallows fillets and small logo marks.
- Use separate tolerances for distance, radius, and angle. A radial residual uses the radius tolerance. A gap uses the distance tolerance. Parallelism, perpendicularity, sweep membership, and angle targets use the angle tolerance.
- Keep the degeneracy and branch checks in the table above when a tolerance is loosened. A looser tolerance does not turn coincident centers into a tangent circle, and it does not turn a zero-length segment into a direction.
- Rank candidates of one relation by residual divided by that relation's tolerance. A ratio below 1 means "plausible at the current tolerance." Review still accepts or rejects. Conflicting plausible candidates are reduced to one accepted encoding before a solve.
- On solver failure, or on a large move away from the input, revert. Blind acceptance is outside this proposal.

### Proposed defaults (not validated)

No project threshold has been calibrated. The earlier vocabulary page mentioned two illustration bands. They are recorded here so the review can see the same examples. They are starting points a later prototype could tune. They are not published thresholds, and they are not measurements.

| Quantity | Illustrative starting point | Status |
| --- | --- | --- |
| Parallel angular residual | on the order of 2° | Unvalidated example from the earlier notes |
| Equal-sweep angular residual | on the order of 1° | Unvalidated example from the earlier notes |
| Distance absolute floor | unset | Future calibration |
| Distance relative scale | unset | Future calibration. Local feature size, not the page diagonal |
| Radius tolerance | unset | Separate from the distance tolerance |
| Other angle tolerances | unset | Separate from the two bands above |
| Solver displacement budget | unset | Future calibration. Until then, revert on a failed or badly moved solve |

Calibration sources, when that work happens: clean renders, synthetic perturbed sketches, real hand drawings, and traced SVG. The earlier page preferred clean renders as the first set. That remains a reasonable first set, and it is not the whole calibration.

Confidence values in the JSON are provenance scores. They are not calibrated probabilities. Nothing in this repository has measured a probability for them.

## Illustrative JSON

Conceptual fields for review. There is no parser and no schema implementation.

The file is self-contained: every referenced shape id exists, subreferences are `start`, `end`, `center`, or `whole`, dimensions carry targets and units, symmetry names its axis, and tangency names a branch or a side plus a contact. `B1` is preserved unsupported geometry and has no constraints. Phase 2 relations are included so their fields are visible; a later phase-1 prompt would omit those types. This is not a solved sketch.

Review values:

- `c_perpendicular_L1_L3` is numerically satisfied and rejected because the accepted 90° angle already states that relationship.
- `c_equal_sweep_A1_A2` is numerically unsatisfied and rejected. The mirror is `c_symmetry_A1_A2`.
- `c_horizontal_L1` is numerically satisfied in this frame and left proposed. Axis alignment of this drawing is not by itself design intent.
- The scores `0.86`, `0.91`, and `0.8` are the illustrative vision scores from the earlier page, carried with the parallel, equal-radius, and line–circle tangent candidates. `confidence_calibrated` is false.
- `c_offset_L1_L2` is the phase-2 encoding of the gap between `L1` and `L2`. Accepting it together with both the parallel candidate and the accepted point–line distance would restate that gap three ways. The point–line distance by itself does not keep `L2` parallel.
- Accepting both `c_midpoint_P1_L1` and `c_point_on_P1_L1` would be redundant: the midpoint of a segment lies on that segment.

```json
{
  "schema": "simplest-path.vocabulary.illustrative",
  "schema_status": "conceptual_not_implemented",
  "example_status": "illustrative_candidates_not_a_solved_sketch",
  "units": { "length": "px", "angle": "deg" },
  "frame": {
    "origin": "page",
    "x": "right",
    "y": "up",
    "zero_degrees": "+x",
    "positive_sweep": "counterclockwise"
  },
  "shapes": [
    { "id": "P1", "type": "point", "at": [100, 60] },
    { "id": "L1", "type": "line", "start": [40, 60], "end": [160, 60] },
    { "id": "L2", "type": "line", "start": [40, 20], "end": [160, 20] },
    { "id": "L3", "type": "line", "start": [160, 60], "end": [160, 140] },
    { "id": "C1", "type": "circle", "center": [100, 100], "radius": 40 },
    { "id": "C2", "type": "circle", "center": [180, 100], "radius": 40 },
    {
      "id": "A1",
      "type": "arc",
      "center": [100, 100],
      "radius": 40,
      "start_deg": 180,
      "sweep_deg": -90
    },
    {
      "id": "A2",
      "type": "arc",
      "center": [100, 100],
      "radius": 40,
      "start_deg": 0,
      "sweep_deg": 90
    },
    {
      "id": "S1",
      "type": "line",
      "role": "symmetry_axis",
      "start": [100, 0],
      "end": [100, 200]
    },
    {
      "id": "B1",
      "type": "bezier",
      "support": "preserved_unsupported",
      "controls": [[10, 10], [30, 40], [50, 10], [70, 40]]
    }
  ],
  "constraints": [
    {
      "id": "c_coincident_L1_L3",
      "type": "coincident",
      "a": { "shape": "L1", "ref": "end" },
      "b": { "shape": "L3", "ref": "start" },
      "phase": "1",
      "provenance": "numeric_check",
      "review": "accepted"
    },
    {
      "id": "c_point_on_P1_L1",
      "type": "point_on",
      "a": { "shape": "P1", "ref": "whole" },
      "b": { "shape": "L1", "ref": "whole" },
      "extent": "finite",
      "phase": "1",
      "provenance": "numeric_check",
      "review": "proposed"
    },
    {
      "id": "c_point_on_P1_C1",
      "type": "point_on",
      "a": { "shape": "P1", "ref": "whole" },
      "b": { "shape": "C1", "ref": "whole" },
      "phase": "1",
      "provenance": "numeric_check",
      "review": "proposed"
    },
    {
      "id": "c_parallel_L1_L2",
      "type": "parallel",
      "a": { "shape": "L1", "ref": "whole" },
      "b": { "shape": "L2", "ref": "whole" },
      "phase": "1",
      "provenance": "vision",
      "confidence": 0.86,
      "confidence_calibrated": false,
      "review": "proposed"
    },
    {
      "id": "c_perpendicular_L1_L3",
      "type": "perpendicular",
      "a": { "shape": "L1", "ref": "whole" },
      "b": { "shape": "L3", "ref": "whole" },
      "phase": "1",
      "provenance": "numeric_check",
      "review": "rejected",
      "review_note": "Numerically satisfied, and redundant with c_angle_L1_L3."
    },
    {
      "id": "c_tangent_L1_C1",
      "type": "tangent",
      "a": { "shape": "L1", "ref": "whole" },
      "b": { "shape": "C1", "ref": "whole" },
      "side": "left_of_direction",
      "contact": { "x": 100, "y": 60 },
      "phase": "1",
      "provenance": "vision",
      "confidence": 0.8,
      "confidence_calibrated": false,
      "review": "proposed"
    },
    {
      "id": "c_tangent_C1_C2",
      "type": "tangent",
      "a": { "shape": "C1", "ref": "whole" },
      "b": { "shape": "C2", "ref": "whole" },
      "branch": "external",
      "contact": { "x": 140, "y": 100 },
      "phase": "1",
      "provenance": "numeric_check",
      "review": "proposed"
    },
    {
      "id": "c_equal_length_L1_L2",
      "type": "equal_length",
      "a": { "shape": "L1", "ref": "whole" },
      "b": { "shape": "L2", "ref": "whole" },
      "phase": "1",
      "provenance": "numeric_check",
      "review": "proposed"
    },
    {
      "id": "c_equal_radius_A1_A2",
      "type": "equal_radius",
      "a": { "shape": "A1", "ref": "whole" },
      "b": { "shape": "A2", "ref": "whole" },
      "phase": "1",
      "provenance": "vision",
      "confidence": 0.91,
      "confidence_calibrated": false,
      "review": "accepted"
    },
    {
      "id": "c_distance_L2_L1",
      "type": "distance",
      "a": { "shape": "L2", "ref": "start" },
      "b": { "shape": "L1", "ref": "whole" },
      "extent": "finite",
      "target": 40,
      "units": "px",
      "phase": "1",
      "provenance": "human",
      "review": "accepted"
    },
    {
      "id": "c_radius_C1",
      "type": "radius",
      "a": { "shape": "C1", "ref": "whole" },
      "target": 40,
      "units": "px",
      "phase": "1",
      "provenance": "human",
      "review": "accepted"
    },
    {
      "id": "c_diameter_C2",
      "type": "diameter",
      "a": { "shape": "C2", "ref": "whole" },
      "target": 80,
      "units": "px",
      "phase": "1",
      "provenance": "human",
      "review": "proposed"
    },
    {
      "id": "c_angle_L1_L3",
      "type": "angle",
      "a": { "shape": "L1", "ref": "whole" },
      "b": { "shape": "L3", "ref": "whole" },
      "target": 90,
      "units": "deg",
      "orientation": "directed_start_to_end_ccw_positive",
      "phase": "1",
      "provenance": "human",
      "review": "accepted"
    },
    {
      "id": "c_concentric_A1_C1",
      "type": "concentric",
      "a": { "shape": "A1", "ref": "center" },
      "b": { "shape": "C1", "ref": "center" },
      "phase": "2",
      "provenance": "numeric_check",
      "review": "proposed"
    },
    {
      "id": "c_midpoint_P1_L1",
      "type": "midpoint",
      "a": { "shape": "P1", "ref": "whole" },
      "b": { "shape": "L1", "ref": "whole" },
      "phase": "2",
      "provenance": "numeric_check",
      "review": "proposed"
    },
    {
      "id": "c_symmetry_A1_A2",
      "type": "symmetry",
      "a": { "shape": "A1", "ref": "whole" },
      "b": { "shape": "A2", "ref": "whole" },
      "axis": { "shape": "S1", "ref": "whole" },
      "correspondence": "start_to_start",
      "phase": "2",
      "provenance": "numeric_check",
      "review": "proposed"
    },
    {
      "id": "c_equal_sweep_A1_A2",
      "type": "equal_sweep",
      "a": { "shape": "A1", "ref": "whole" },
      "b": { "shape": "A2", "ref": "whole" },
      "phase": "2",
      "provenance": "numeric_check",
      "review": "rejected",
      "review_note": "Signed sweeps are -90 and +90 degrees. The mirror relationship is c_symmetry_A1_A2."
    },
    {
      "id": "c_offset_L1_L2",
      "type": "offset",
      "a": { "shape": "L1", "ref": "whole" },
      "b": { "shape": "L2", "ref": "whole" },
      "target": 40,
      "units": "px",
      "side": "right_of_direction",
      "phase": "2",
      "provenance": "numeric_check",
      "review": "proposed",
      "review_note": "Accepting this together with both c_parallel_L1_L2 and c_distance_L2_L1 restates one gap three ways. The point-line distance alone does not keep L2 parallel."
    },
    {
      "id": "c_horizontal_L1",
      "type": "horizontal",
      "a": { "shape": "L1", "ref": "whole" },
      "phase": "2",
      "provenance": "numeric_check",
      "review": "proposed",
      "review_note": "True in this page frame. Left unaccepted because page alignment is not design intent by itself."
    }
  ]
}
```

`side` is relative to the directed segment in `a`, in the declared frame. `left_of_direction` is the counterclockwise side (`L1` points in `+x`, and `C1` lies in `+y`). `right_of_direction` is the clockwise side, where `L2` lies. Contact coordinates are the candidate contact used by the check, not an extra solved point.

## Corrections to the earlier vocabulary page

The body above is the vocabulary to review. These are the places the earlier page is not followed literally.

1. An arc was described as "the same as a circle sector." The entity is a circular curve with center, radius, start angle, and signed sweep. It is not a filled sector.
2. The sample JSON used placeholder coordinates `[x, y]` and referred to `L2` and `A2`, which were not among the shapes. The sample in this file defines every referenced shape and uses concrete coordinates.
3. Bands "under about 2 degrees" and "about 1 degree" were written as check examples. They stay illustrative defaults. They are not validated thresholds.
4. The phase column was not labeled as a proposal. Horizontal, vertical, and concentric are called inexpensive and high value for technical sketches, and symmetry high value for logos, as scoping judgments. They are not a benchmark conclusion.
5. `point_on` was limited to lines and circles. Arcs are included. The arc test is the radial residual and signed-sweep membership.
6. A midpoint check that only compared distances to the two endpoints would accept the whole perpendicular bisector. The residual is the distance to `(a + b) / 2`.
7. Tangency was a pair of entity types without a finite-contact test or a branch. Radius sum, radius difference, or line distance is not a complete test. SolveSpace's documented tangent also assumes a shared endpoint; interior contact is in scope here.
8. Coincident centers and equal circles were not distinguished from ordinary tangent circles. They are concentric and equal-radius cases.
9. Confidence figures were bare numbers. They are uncalibrated provenance scores unless a later measurement says otherwise.
10. Bézier input was "kept as input only until a snap layer exists," which invites a later silent replacement. Unsupported curves are preserved or marked unsupported, and they are not approximated.
11. `length` was a separate dimensional row. It is the endpoint distance of a segment.
12. `fix` / `lock` was only a late user-facing constraint. User-facing lock stays deferred. A solver's gauge anchor is still required implementation work, and it is not design intent.
13. Scope was "logos and clean vector art." The scope here is logos and simple technical sketches, with PNG and rough traced SVG sharing one model, matching the project intent.
14. Clean renders were the preferred test set. Calibration, when it happens, also includes synthetic perturbed sketches, real hand drawings, and traced SVG.
15. `offset` appeared in the phase-2 prose and was missing from the phase table. It is in the table now.

## Limitations and open decisions

Limitations of this document:

- No checker, solver, importer, exporter, or review UI is specified as code.
- Field names in the JSON are a conceptual sketch. They are not a compatibility contract.
- No tolerance has been fit to data. Using the 2° or 1° illustrations as if they were measured would be a misuse of this file.
- References describe other systems. They are not results for Simplest Path.
- The sample geometry is hand-built so the listed residuals can be checked by hand. It is not a traced drawing and not evidence that a vision model would propose these relations.

Open decisions, all future work:

- The normalization map from PNG pixels and SVG user units into the declared frame, including y-down rasters.
- Values for the absolute floor, the local relative scale, the radius tolerance, the remaining angle tolerances, and the displacement budget.
- Whether a line–line `distance` constraint implies `parallel`, or whether parallel is always a separate accepted relation.
- Whether `equal_sweep` should ever compare absolute spans. The proposal is signed sweeps, so mirrors go through `symmetry`.
- Whether a sweep of exactly ±360° is rejected, rewritten as a circle, or preserved as a degenerate arc. The proposal is: full turns are circles.
- How symmetry names endpoint correspondence when one arc's parameterization is not the mirror-order of the other.
- How preserved Bézier and ellipse data are stored later: a passthrough SVG fragment, or explicit control points as in `B1`.
- How an offset arc inherits sweep.
- Which internal gauge anchor a solver uses (a pinned point and a direction, a pinned centroid and a reference angle, or something else). The user does not have to supply `fix` / `lock` for that.
- Whether rectangle and polygon helpers are expanded before inference or kept as a group of segments.
- Which solver, if any. The earlier page named SolveSpace and planegcs (FreeCAD's PlaneGCS) as candidates. Neither is selected, and neither is a dependency of this repository.
- How vision scores become calibrated probabilities, if they ever do.
- The phase-1 subset to show a vision model. The earlier page wanted that prompt kept small. The prompt is not part of this change.

Out of scope, listed so they are not added quietly: refraction (Snell's law), sketch patterns, 3D workplanes, and silent approximation of splines.

## Future work

Explicitly not in this change:

- A deterministic candidate checker that computes the residuals in the verification table.
- A human review surface for accept, reject, and edit.
- A solver, with gauge anchoring, redundancy and inconsistency reporting, and revert on failure or large displacement.
- Calibration of tolerances on clean renders, synthetic perturbed sketches, real hand drawings, and traced SVG.
- A decision on the conceptual JSON fields, and only then a parser.
- SVG export of cleaned lines, circles, and arcs together with the editable constraint set.

## References

Cited so a reviewer can see where the vocabulary was compared. These links are not accuracy claims, and this document does not report a reproduction of any of these systems.

- [SolveSpace reference](https://solvespace.com/ref.pl). Constraint sections used as comparison points: Distance / Diameter, Angle, Horizontal / Vertical, On Point / Curve / Plane, Equal Length / Radius / Angle, Length Ratio, Length Difference, At Midpoint, Symmetric, Perpendicular, Parallel / Tangent, and Lock Point Where Dragged. Also the entity descriptions of line segment, rectangle, circle, and arc. Facts taken from that page and accounted for above: an angle is stored in 0°–180° with an explicit supplementary-angle command; parallel/tangent for curves requires an existing shared endpoint; horizontal and vertical are defined by the active workplane and are described as simple to solve; rectangles are four line segments.
- [FreeCAD Sketcher Workbench](https://wiki.freecad.org/Sketcher_Workbench). Official workbench documentation. The current English documentation lists geometric constraints Coincident, Point on object, Horizontal, Vertical, Parallel, Perpendicular, Tangent or collinear, Equal, Symmetric, and Block; dimensional constraints Distance, Radius, Diameter, Angle, and Lock; and Refraction (Snell's law). The [coincident tool page](https://wiki.freecad.org/Sketcher_ConstrainCoincident) says that tool also creates a concentric constraint between circles, arcs, and ellipses by making their centers coincident, and its usage includes elliptical arcs. This vocabulary splits coincident from concentric, and tangent from collinear, on purpose. The workbench page describes a fully constrained sketch as one with no remaining freedom, which is the CAD aim distinguished from [minimal representation](#minimal-representation). Its best-practices text says horizontal and vertical distance constraints are computationally cheaper than the general length constraint, and lists horizontal and vertical constraints among the ones to prefer. That guidance is FreeCAD's, and the phase column here stays a proposal. The constraint list was checked against the English documentation sources for that workbench.
- [SketchGraphs constraint checks](https://princetonlips.github.io/SketchGraphs/_autosummary/sketchgraphs.data.constraint_checks.html). A geometric checker for an already built sequence. The module states that only checking is implemented and that it is not a solver. The documented check functions include `coincident`, `concentric`, `equal`, `horizontal`, `midpoint`, `parallel`, `perpendicular`, `tangent`, and `vertical`. No tolerance table is cited from that page, and none is invented here.
- [SketchGraphs repository](https://github.com/PrincetonLIPS/SketchGraphs). Onshape-derived sketches as constraint graphs. `ConstraintType` includes, among other members, `Coincident`, `Distance`, `Horizontal`, `Parallel`, `Vertical`, `Tangent`, `Length`, `Perpendicular`, `Midpoint`, `Equal`, `Diameter`, `Offset`, `Radius`, `Concentric`, `Fix`, `Angle`, `Circular_Pattern`, and `Linear_Pattern`. Local references can point at `.start`, `.end`, or `.center`. Numerical constraints carry parameters such as angle, clockwise, length, and half-space. That public shape is why the illustrative JSON stores subreferences, targets, orientation, and tangency branch explicitly. It is not a claim that these residuals match Onshape or SketchGraphs, and it is not a published accuracy number.
