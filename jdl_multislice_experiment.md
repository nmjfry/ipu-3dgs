# The multiSlice gather experiment

This note documents an experiment in replacing the NEWS Manhattan routing
with a dynamic gather, in which each tile pulls the Gaussians it needs
directly from the tiles that hold them. The aim was to test whether this
would reduce frame time and remove the flicker caused by channel saturation.

Summary of the outcome: gather transport is fast (0.24 ms for the full
per-frame working set), but computing which global indices each tile needs
(discovery) does not fit in tile-local memory, so the implementation in this
repository performs discovery on the host instead. The path is enabled with
`--gather-mode multislice`. It was not used for the paper's results; the
default `--gather-mode news` is the pipeline described in the paper.

## Candidate mechanisms

Two mechanisms were considered.

1. **JDL (jit dynamic lookup).** A Graphcore Research prototype in which a
   single receiver tile pulls a contiguous slice from any sender tile, with
   the source tile and offset chosen at runtime. It is unsuitable here: a
   tile that holds data may not also be a receiver, and this restriction is
   enforced by an assertion inside a precompiled binary that cannot be
   patched. In this renderer every tile both holds Gaussians and needs to
   receive them. Routing through the roughly 32 spare tiles would require
   about 45 batched rounds per frame, which is not viable.

2. **`popops::multiSlice` (standard Poplar).** A planned gather that pulls M
   rows from an [N, width] table using a runtime offsets tensor, with no
   sender or receiver restriction. This is the mechanism the experiment
   uses.

## Discovery is the limiting factor

Either mechanism only moves a Gaussian once a tile knows its global index.
In the NEWS pipeline, routing itself performs discovery: a Gaussian walks
towards its anchor tile and propagates to the neighbours it overlaps. A
gather has no such side channel, so each tile must compute the indices of
all Gaussians overlapping its screen region by some other means.

The naive approach, broadcasting every Gaussian's 2D footprint to every
tile, needs roughly 44,000 x (8 B mean + 4 B radius + 4 B index), about
700 KB per tile. This exceeds the 624 KB tile budget, so a full metadata
AllGather does not fit. This memory constraint is the reason the paper uses
local message passing.

Three discovery schemes were considered: exchanging footprints only between
nearby screen regions (a spatial hash), computing the assignment on the
host, and a two-level multiSlice that gathers compact metadata before the
full structs. The code here implements host-computed assignment, which
isolates the gather performance and gives immediate convergence with no
flicker, at the cost of moving part of the pipeline off the chip.

## Stage 1: transport microbenchmark

`tools/multislice_bench.cpp` (build target `multislice_bench`) measures the
raw `popops::multiSlice` gather cost for a representative workload: it
gathers `numTiles x perTile` rows of 15 floats (one `Gaussian3D`) from a
`numGaussians x 15` table, amortised over many repeats.

```bash
# in the container build directory
ninja multislice_bench
./multislice_bench                 # defaults: 44000 entries, 400 per tile, 1440 tiles
./multislice_bench 44000 400       # numGaussians perTile
```

Result (SDK 3.3, C600, June 2026):

```
table:   44000 entries x 15 floats
lookups: 576000 (1440 tiles x 400)   # full per-frame working set
per gather: 0.238 ms   (32.96 MiB per gather, about 138 GB/s)
```

The achieved bandwidth is close to the exchange-fabric peak, and the
per-gather time is roughly 40 times smaller than the 9 ms per-frame compute
cost. Transport is therefore not the bottleneck.

A second measurement checked output placement: gathering and then copying
the result into a tensor pinned at 400 rows per tile (the layout the blend
stage expects) costs 0.2443 ms per frame against 0.2387 ms for the gather
alone. Forcing exact per-tile placement therefore costs about 5.6
microseconds, which is negligible, and a single global multiSlice is the
right shape rather than per-tile gathers.

## Stage 2: renderer integration

The gather path is integrated behind `--gather-mode news|multislice`
(default `news`; the NEWS path is unchanged). Discovery is host-assisted:
the host projects the Gaussians and assigns them to tiles, then streams the
offsets and counts down each frame; the device gathers, pins, projects and
blends.

The components are:

- `include/splat/gather_discovery.hpp`: `computeGatherOffsets()`, the host
  discovery step.
- `codelets/splat/codelets.cpp`, `GatherProjectVertex`: projects and sorts
  the gathered Gaussians (the projection stage of `RouteVertex` without the
  routing), writing `gaus2D` in the same format so `BlendVertex` is
  unchanged.
- `src/splat/ipu_rasteriser.cpp`, `buildGatherPath()` and
  `executeGather()`: the sliceable table, the per-frame offsets and counts
  streams, and the gather, pin, project, blend sequence.
- `src/main/splat.cpp`: the `--gather-mode` flag.

To run:

```bash
./build/src/main/splat --input scene.ply --device ipu --gather-mode multislice --ui-port 5000
```

and compare frame time and flicker against `--gather-mode news` at the same
pose.

## Conclusion

Transport is cheap and output mapping is essentially free; the unsolved
problem is discovery within the per-tile memory budget. Host-assisted
discovery works and removes flicker, but it moves part of the pipeline off
the chip, which is contrary to the aim of keeping all state on-chip, and it
was not developed further. On current tooling this result supports the
paper's choice of local message passing; an on-chip discovery scheme (such
as the spatial-hash exchange above) remains the open problem for a fully
resident gather pipeline.
