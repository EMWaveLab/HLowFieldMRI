# Halbach MRI Field Simulator

An interactive, browser-based simulator for analysing Halbach cylinder magnet arrays used in low-field MRI systems. The tool computes the magnetic field produced by an arbitrary configuration of permanent-magnet rings, visualises it in real time as 2D field maps and a 3D scene, and reports homogeneity metrics over a cylindrical ROI.

No server, no installation — open the HTML file in any browser.

---

## Background

### Halbach arrays in MRI

Conventional MRI scanners use superconducting solenoids to generate a strong, highly uniform static field **B₀**. A Halbach cylinder is an alternative geometry in which permanent magnets are arranged around a cylindrical bore so that their magnetisation vectors rotate by twice the angular position. This rotation constructively reinforces the field inside the bore while cancelling it almost completely outside — making Halbach arrays a geometry for portable and low-field MRI research.

The figure of merit for MRI is **field homogeneity**: the fractional variation of |**B**| over the imaging volume, expressed in **ppm** (parts per million). Even small ppm variations cause image distortion and chemical-shift artefacts. Placing multiple Halbach rings at optimised axial positions and with varying numbers of magnets per ring is the primary lever to reduce this inhomogeneity.

### Physical model

Each magnet is modelled as a **uniformly-magnetised rectangular cuboid**. The field is computed with the exact closed-form solution derived from the Coulombian model:

```
B(r) = (μ₀/4π) · ∇ × ∫ M × ∇(1/|r−r'|) dV'
```

The implementation follows the **magpylib** analytical formulation for cuboid sources.

For a ring with **N** magnets of remanence polarisation **B_r** at radius **R** and axial position **z**, each magnet is a cube of side **D** rotated so that:

- its centre is at angle `α = 2πi/N` on the ring circle  
- its magnetisation vector points at angle `2α` (the Halbach condition)

The total field at any point is the superposition of contributions from all magnets in all rings.

---

## Features

| Category | Detail |
|---|---|
| **Geometry** | Arbitrary number of Halbach rings; each ring has independently configurable magnet count N, radius R, and axial offset z |
| **Global parameters** | Remanent polarisation B_r (0.1 – 2.0 T), magnet side length D (4 – 24 mm), grid resolution n (40 – 160) |
| **3D visualisation** | Interactive Three.js scene showing all magnet cuboids with polarisation arrows; drag to rotate, scroll to zoom |
| **Field maps** | Two orthogonal 2D slice views: XZ plane (longitudinal cross-section) and XY plane (transverse cross-section) |
| **Display modes** | Intensity heatmap, magnetic field lines (RK4 streamlines), dB (logarithmic) scale — toggled per view simultaneously |
| **Post-processing** | 2D Gaussian smoothing with adjustable σ (0 = off, up to 5) applied before rendering |
| **Colormaps** | coolwarm, plasma, viridis, hot, jet |
| **Field lines density** | Adjustable seed density for the streamline renderer |
| **Slice navigation** | Independent slice-position sliders for each view; full-width sliders spanning the plot for fine-grained control |
| **Zoom & pan** | Scroll-wheel zoom and drag-to-pan on each 2D canvas; double-click to reset |
| **ROI analysis** | Mean \|B\| (mT), standard deviation (mT), homogeneity (ppm), and voxel count over a cylinder of radius 0.5·R_min spanning the ring stack |
| **Parallel computation** | Field is computed on all available CPU cores via Web Workers; each worker handles a subset of z-layers |
| **One-shot compute** | Full 3D grid (n³ voxels) is computed once; slice navigation only reads from the stored volume — zero recomputation |

---

## Getting started

1. Clone or download the repository.
2. Open `index.html` directly in a browser (no web server required).
3. Adjust the ring configuration in the left panel.
4. Click **▶ COMPUTE FIELD**.
5. Explore the field maps with the slice sliders and display-mode buttons.

> **Tip:** Start at the default grid resolution (40) for a fast first look, then increase to 80–120 to increase the map quality.

---

## Interface guide

### Left panel — global & ring controls

| Control | Effect |
|---|---|
| **Polarization (T)** | Remanent magnetisation B_r of each magnet cube |
| **Magnet (mm)** | Side length D of the cubic magnets |
| **Grid res** | Number of grid points n per axis; the 3D field is sampled on an n × n × n Cartesian grid |
| **+ add ring** | Appends a new ring to the stack with sensible defaults |
| **RING card** | Per-ring controls: N (number of magnets), R (radius in cm), Z offset (axial position in cm); click ✕ to delete |
| **▶ COMPUTE FIELD** | Launches parallel computation; badge shows "COMPUTING k/N" during execution and "READY" on completion |

### Shared display controls (top bar above slice views)

| Control | Effect |
|---|---|
| **Intensity** | Toggles the colour-mapped heatmap overlay |
| **Lines** | Toggles RK4 magnetic field-line streamlines |
| **dB** | Switches the colour scale to decibels: 10·log₁₀(\|B\|) |
| **smooth σ** | Gaussian post-processing blur; σ = 0 shows raw computed values, higher σ gives smoother colour transitions |
| **density** | Number of streamline seeds; affects both views simultaneously |
| **colormap** | Selects the colour palette for both views |

All display controls apply to **both** slice views simultaneously.

### Slice views

- **XZ view** (top): longitudinal cross-section; the red dashed line marks the position of the XY slice.
- **XY view** (bottom): transverse cross-section; the red dashed line marks the position of the XZ slice.
- The **y-slice** and **z-slice** sliders select which plane to display; they span the full width of the plot for fine control.
- Yellow dashed lines show the ROI cylinder boundary projected onto each plane.
- The colour bar below each plot shows the min and max values in T (or dB).

### 3D view (top-left)

Displays all magnet cuboids coloured by ring index. Red arrows on each magnet show the polarisation direction (the rotating Halbach pattern). Drag to orbit, scroll to zoom.

### ROI statistics card (bottom-left)

Computed from all voxels inside a cylinder of radius `0.5 · R_min` and axial extent `[z_min_ring, z_max_ring]` — a conservative estimate of the usable bore volume.

| Metric | Description |
|---|---|
| **mean \|B\| mT** | Average field strength inside the ROI |
| **std mT** | Standard deviation of \|B\| inside the ROI |
| **inhomog. ppm** | `(std / mean) · 10⁶` — the primary MRI quality figure |
| **voxels sampled** | Number of grid points inside the cylinder |

---

## Extending the simulator

**Adding a new colormap:** append an entry to the `CMAPS` object in the script. Each entry is an array of 10 `[R, G, B]` control points; the renderer interpolates linearly between them.

**Adding a new display mode:** add a key to `S.plots.xy.modes` / `S.plots.xz.modes`, add a button with `data-shared-m="<key>"` in `#slices-hdr`, and add the corresponding draw branch inside `drawSlice`.

**Changing the ROI geometry:** edit the `Rcyl` and z-range logic in the `computeField` callback that runs after all workers finish.

**Exporting data:** `S.field.Bm3` is a `Float32Array` of shape `[n, n, n]` (C-order, z-major). You can serialise it from the browser console with `JSON.stringify(Array.from(S.field.Bm3))` or use a `Blob` download button.

---

## Limitations

- Magnets are assumed to be perfectly uniformly magnetised cubes with no demagnetisation correction.
- The Halbach rotation angle is fixed at `2α` (pure dipole mode, k=2); higher-order modes are not implemented.
- The simulator runs entirely in the browser; very large configurations with many magnets per ring will increase worker computation time proportionally.

---

## License

MIT — free to use, modify, and redistribute with attribution.
