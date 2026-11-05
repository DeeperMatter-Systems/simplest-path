# simplest-path

Sketch to clean vector geometry: infer shapes and constraints, verify with numeric checks, solve to a minimal set.

## Intent

**Input:** a raster sketch or a rough traced vector (PNG / SVG).

**Output:** clean geometry (lines, arcs, circles) with inferred constraints (parallel, perpendicular, tangent, equal, and related), verified by numeric tolerance checks and solved down to a minimal representation.

## Interfaces planned

- Sketch-input interface (draw or upload a rough sketch)
- Traced-SVG cleanup interface (take a rough vector trace and constrain / simplify it)

Both paths should converge on the same constrained geometry model.
