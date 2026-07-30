# Actin Bundle Analysis (cryo-ET)

A 4-step Python pipeline to separate individual actin filaments from a binary cryo-ET segmentation and classify them as **bundle** (tightly packed, parallel) vs. **mesh/isolated**, based on local orientation and neighbor distance — following a Jasnin-et-al.-style criterion (close, parallel neighbors from *other* filaments).

## Pipeline overview

| Step | Notebook | Input | Output |
|------|----------|-------|--------|
| 1 | `STEP1_Watershed.ipynb` | Binary actin segmentation (`.mrc`) | Per-slice watershed labels (`*_watershed_*.mrc`), optionally a manually-edited mask (`*_edited_v*.mrc`) |
| 2 | `STEP2_Skeletonization_PerFilament.ipynb` | Watershed labels (`*_watershed_*.mrc`) | Per-fragment skeleton, 1-voxel wide (`*_skeleton.mrc`) |
| 3 | `STEP3_Bundle_Scoring.ipynb` | Skeleton volume (`*_skeleton.mrc`) | Per-point scoring table (`*_bundle_scoring.csv`), classification volume (`*_bundle_classification.mrc`) |
| 4 | `STEP4_Bundle_Render.ipynb` | Original segmentation + scoring table (`.csv`) | Colored render volumes (`*_bundle_render.mrc`, `*_bundle_render_filled.mrc`) for figures |

Each step reads the previous step's saved output, so the notebooks can be run independently (or resumed) as long as the expected file is present in the output folder.

---

## Step 1 — Watershed segmentation

Separates the single binary actin blob into individual filaments, **slice by slice, independently** (no linking between Z slices at this stage):

1. Light 2D erosion (cleanup only, not full separation).
2. Distance transform + Gaussian smoothing to avoid spurious local maxima along straight filaments.
3. `peak_local_max` → seeds → `skimage.segmentation.watershed` on the *original* (non-eroded) mask/distance, so no voxels are lost.
4. 3D visual review in Napari; manual correction (erasing fusion voxels) directly on the binary mask, with versioned saves (`*_edited_v1.mrc`, `*_edited_v2.mrc`, …) so you can resume a cleanup session without recomputing.

**Note:** filament IDs are *not* consistent across Z slices at this stage — that's deliberate; identity linking is skipped (see Step 2 rationale).

Key parameters: `min_distance`, `smooth_sigma`, `seed_erosion_radius` — tune by inspecting over-/under-segmentation in the 3D viewer.

---

## Step 2 — Per-filament skeletonization

Rather than skeletonizing the whole volume at once (which failed once filaments touch/merge into one blob), each watershed fragment is skeletonized **individually**, in its own cropped bounding box:

1. **Relabel pass**: because watershed doesn't link IDs across Z, the same numeric ID can appear in two unrelated, far-apart fragments. A single connected-components pass (`scipy.ndimage.label`, 26-connectivity) splits any ID that actually spans disconnected regions, so each real fragment gets a unique ID with a tight bounding box.
2. **Bounding-box diagnostic**: flags abnormally large fragments (possible bad merges) before running skeletonization on the full set.
3. `skimage.morphology.skeletonize` is run per fragment, on its own padded crop, then written back into a full-size skeleton volume — no fragment can "leak" into a neighbor's skeleton.
4. Sanity check: skeleton-voxel-count / Z-extent ratio per fragment (should be ≈1.0 for a clean, unbranched filament).

**Design decision:** cross-slice identity linking is intentionally skipped — for the bundle-scoring metric (point-to-point distance + orientation), a filament broken into several fragments still yields valid fragments with correct local orientation; two fragments that are really the same filament are simply counted as close parallel neighbors of each other, same as a real bundle would be.

---

## Step 3 — Bundle scoring

For every skeleton point:

1. **Local orientation**: PCA/SVD on nearby points (`WINDOW_VOXELS` radius) *restricted to the same fragment* — points from fragments too short for a reliable window are marked as having no orientation.
2. **Neighbor search**: `cKDTree.query_pairs` (vectorized) finds all point pairs within `DIST_THRESHOLD_NM`, excludes same-fragment pairs and pairs without valid orientation, and computes the angle between orientations.
3. **Classification**: a point is `is_bundle = True` if it has at least `N_MIN_PARALLEL` neighbors (from *other* fragments) within the distance and angle thresholds.

**Current parameters** (calibrated by a prior parameter sweep, confirmed visually):

| Parameter | Value |
|-----------|-------|
| `WINDOW_VOXELS` (local PCA window) | 5 voxels (~6.2 nm) |
| `DIST_THRESHOLD_NM` | 10 nm |
| `ANGLE_THRESHOLD_DEG` | 15° |
| `N_MIN_PARALLEL` | 5 |

Output: a per-point CSV (`z_voxel, y_voxel, x_voxel, fragment_id, orientation_xyz, n_parallel_neighbors, is_bundle`) and a classification `.mrc` volume (0 = background, 1 = no-bundle, 2 = bundle).

---

## Step 4 — Bundle render

Reconstructs a colored visualization for figures:

1. Loads the original binary segmentation (as semi-transparent context) and the Step 3 scoring table.
2. Rebuilds a labeled skeleton volume from the table (1 = no-bundle, 2 = bundle, 3 = unclassified — orientation invalid / fragment too short) and displays it in Napari (cyan = bundle, red = no-bundle, gray = unclassified).
3. **Optional**: propagates the skeleton classification to the *full filament thickness* (not just the thin skeleton) via nearest-neighbor (`cKDTree`) — useful when you want the whole segmented volume colored, not just the 1-voxel-wide skeleton line. This step is slower, scaling with the number of voxels in the full segmentation.

---

## Requirements

```bash
pip install numpy scipy scikit-image pandas mrcfile napari
```

Python 3.10+ (developed/tested on 3.11).

## Input data

Binary actin segmentation volumes in `.mrc` format (same voxel space/size as the source tomogram), e.g. from Amira, Dragonfly, or a deep-learning segmentation model.

> ⚠️ Paths throughout the notebooks are hardcoded to local folders (`C:\PhD\Actin_ET\...`). Update `input_file` / `output_dir` at the top of each notebook before running on a different dataset. A couple of cells also reference a fixed filename fragment (e.g. `Pos16_actin_...`) left over from the dataset used during development — check and rename as needed for your own sample.

## Not included

Raw `.mrc` volumes and intermediate results (watershed labels, skeletons, scoring CSVs) are not part of this repository — keep them out of version control (see `.gitignore`).
