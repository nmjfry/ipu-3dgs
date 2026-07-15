# Rendering 3D Gaussians on a Graph Processor

Code release for the EGSR 2026 paper
[*Rendering 3D Gaussians on a Graph Processor*](https://doi.org/10.2312/sr.20261017).

This is a forward-only 3D Gaussian Splatting renderer that runs entirely in
on-chip SRAM on a Graphcore IPU. The framebuffer is partitioned across the
IPU's tiles, each of which owns a rectangular screen-space region. Projected
Gaussians are routed to their destination tiles via Manhattan-distance hops on
a north-east-west-south (NEWS) grid, then propagated to neighbouring tiles
whose regions they overlap. Each tile independently sorts and alpha-blends its
locally stored Gaussians, with no access to global memory. Rendered frames are
streamed over TCP to a remote viewer.

The viewer is a separate repository:
[remote_render_ui](https://github.com/nmjfry/remote_render_ui).

## Repository layout

| Path | Contents |
|---|---|
| `src/main/splat.cpp` | Server entry point: PLY loading, UI server, render loop |
| `src/splat/` | Host-side pipeline: camera, projection, graph construction |
| `codelets/splat/codelets.cpp` | On-tile code: routing (RouteVertex) and blending (BlendVertex) |
| `include/splat/` | Gaussian maths (Cov3D/Cov2D), viewport mapping, geometry |
| `include/tileMapping/` | Tile configuration and NEWS channel wiring |
| `tools/` | Scene download, benchmarking, plotting and GPU reference scripts |
| `tests/` | Unit and pipeline tests against the reference 3DGS maths |

## Requirements

- A Graphcore IPU (tested on Mk2 / C600) with **Poplar SDK 3.3**.
- CMake, Ninja, Boost and OpenCV in the build environment.

The most reliable route is the tested Docker image:

```bash
git clone https://github.com/markp-gc/docker-files.git
docker build -t $USER/poplar_3.3_dev --build-arg UNAME=$USER \
  --build-arg UID=$(id -u) --build-arg GID=$(id -g) \
  docker-files/graphcore/poplar_dev
```

Launch it with `gc-docker` so the IPU devices are visible inside the
container (source your host's Poplar SDK first to get `gc-docker`).

## Build

```bash
git clone --recursive <this-repo-url>
cd <repo>
mkdir build && cd build
cmake -G Ninja ..
ninja
```

## Scenes

No scene data is included. `tools/fetch_dylanebert_3dgs.py` downloads small
pre-trained 3DGS scenes from the Hugging Face dataset `dylanebert/3dgs` and
strips higher-order spherical harmonics to produce DC-only `.ply` files that
fit the loader. Any DC-only 3DGS `.ply` (including Gaussian-Splatting-SLAM
output) should work.

## Run

Start the render server on the IPU machine:

```bash
./build/src/main/splat --input data/scene.ply --device ipu --ui-port 5000
```

Then connect the [viewer](https://github.com/nmjfry/remote_render_ui) from
your own machine:

```bash
./build/remote-ui-imgui --host <ipu-machine> --port 5000
```

If the server is behind SSH, forward the port first:
`ssh -L 5000:localhost:5000 <user>@<ipu-machine>` and connect to
`localhost`.

Selected server options:

| Flag | Effect |
|---|---|
| `--device ipu\|cpu` | Render device (CPU path is point-only, for debugging) |
| `--from-pose <json>` | Start from a saved camera pose |
| `--orbit <deg>` | Orbit the camera automatically |
| `--play-path <traj>` | Play back a recorded camera trajectory |
| `--benchmark N` | Headless benchmark: N routing substeps per zoom level |
| `--device-loop` | Device-side render loop (no per-frame host round trip) |

## Benchmarking

`--benchmark N` writes `benchmark_profile.csv` with per-phase timing
(route / blend / exchange), an on-tile cycle-counter breakdown across all
tiles, and convergence tracking. Plot with `tools/plot_profile.py`. For power
measurements, run via `tools/benchmark_with_power.sh` (requires
`GCDA_MONITOR=1` for `gc-monitor` sensor access).

## GPU reference renderer

`tools/render_gpu_dgr.py` renders identical poses with the original CUDA
rasteriser ([diff-gaussian-rasterization](https://github.com/graphdeco-inria/diff-gaussian-rasterization))
for quality and timing comparisons. It requires a CUDA-capable GPU and a
Python environment with that package installed; it is not needed to run the
IPU renderer.

## Experimental: multiSlice gather

The flags `--gather-mode multislice`, `--device-discovery` and
`--gather-cap` select an experimental alternative to NEWS routing based on
Poplar `multiSlice` gathers. This path is retained for reference but was not
used for the paper's results; the default (`--gather-mode news`) is the
pipeline described in the paper. Notes on the experiment are in
`jdl_multislice_experiment.md`.

## License

Unless stated otherwise the content of this repository is available under the
terms in [LICENSE](./LICENSE). Submodules under `external/` are licensed under
their own terms.

## Citation

```bibtex
@inproceedings{10.2312:sr.20261017,
  booktitle = {Rendering 2026 Symposium Track},
  editor    = {Gkioulekas, Ioannis and Jarabo, Adrian},
  title     = {{Rendering 3D Gaussians on a Graph Processor}},
  author    = {Nicholas Fry and Ignacio Alzugaray and Mark Pupilli and Paul H. J. Kelly and Andrew J. Davison},
  year      = {2026},
  publisher = {The Eurographics Association},
  ISSN      = {1727-3463},
  ISBN      = {978-3-03868-320-9},
  DOI       = {10.2312/sr.20261017}
}
```
