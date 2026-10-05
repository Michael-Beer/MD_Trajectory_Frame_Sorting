# MD Trajectory Frame Filter (cpptraj)

A bash wrapper around [cpptraj](https://amberhub.chpc.utah.edu/cpptraj/) (AmberTools) that identifies MD frames in which a water molecule is positioned for nucleophilic attack on a carbonyl carbon. For each frame it finds the water closest to a reference atom, measures its distances and its **Bürgi–Dunitz (BD) angle** relative to the carbonyl, and then extracts only the frames that satisfy all user-defined geometric criteria.

The example configuration is set up for OXA-163 in complex with cefiderocol, but the atom masks and thresholds are all configurable.

## What it does

1. Loads all replicate trajectories with a shared topology.
2. Uses `closest` to keep, for every frame, the single water nearest to a reference atom (`closest_mask`). This writes a reduced trajectory and matching topology.
3. On the reduced trajectory, calculates:
   - the distance from the selected water oxygen to `closest_mask` (`closestdist1`)
   - the distance from the selected water oxygen to `closest_mask2` (`closestdist2`)
   - the BD angle: water O → carbonyl C → carbonyl O (`BDangle`)
   - normalised histograms of both distances (0–10 Å, 100 bins)
4. Reloads the **original, full-system** trajectories and uses `filter` to keep only frames where the BD angle and both distances fall within their allowed ranges.
5. Writes the surviving frames to a new trajectory.

## Requirements

- AmberTools with `cpptraj` (the script loads `apps/amber/24.tools.24` via the environment modules system; change this line for your cluster or install)
- A bash shell

## Input layout

```
<in_directory>/
├── system.parm7
├── Run1/prod1.nc
├── Run2/prod2.nc
└── Run3/prod3.nc
```

The same topology (`system.parm7`) is used for every run.

## Usage

Edit the variables at the top of the script, then run:

```bash
bash hydrolysis_frame_filter.sh
```

The script creates `out_directory` and writes all results there. For long trajectories, submit it through your cluster scheduler instead of running it interactively.

## Configuration

| Variable | Example | Description |
|---|---|---|
| `in_directory` | `/path/to/OXA163_cefiderocol/` | Directory containing `system.parm7` and `RunN/` folders |
| `out_directory` | `/path/to/needle_hydrolysis_2/` | Where results are written (created by the script) |
| `runs` | `(1 2 3)` | Replicate numbers, expanded to `Run${run}/prod${run}.nc` |
| `dataset_name` | `163_cef_hydrolysis_A` | Label appended to output filenames |
| `BD_residue` | `50` | Residue containing the carbonyl group being attacked |
| `BD_atoms` | `(O C30 O33)` | Atom names for the angle: water O (taken from the closest-water residue), carbonyl C, carbonyl O |
| `BD_range` | `(105 109)` | Allowed BD angle range (degrees) |
| `closest_mask` | `53@OQ2` | Reference atom used to select the closest water (`residue@atom`) |
| `closest_range` | `(0 3)` | Allowed water O to `closest_mask` distance (Å) |
| `closest_mask2` | `50@C30` | Second reference atom (the electrophilic carbon) |
| `closest_range2` | `(0 4)` | Allowed water O to `closest_mask2` distance (Å) |

The script also contains commented-out options for a dihedral filter (`dihedral_*`) and a second-water distance filter (`closest_mask3`, `closest_range_wat1/2`). Uncomment and wire these into the `filter` command if needed.

## Output

All files are written to `out_directory`.

| File | Contents |
|---|---|
| `closeststats_<dataset>.dat` | Per-frame information on the closest water (from `closestout`) |
| `<closest_mask>_closest1.nc` | Reduced trajectory: solute plus the single closest water per frame |
| `<closest_mask>_closest1.system.parm7` | Topology matching the reduced trajectory |
| `stats_<dataset>.dat` | Time series of both distances and the BD angle, plus filter output |
| `hist_<dataset>.dat` | Normalised histograms of both distances |
| `filteredframes_<dataset>.nc` | Frames from the original trajectories that pass all filters |

`filteredframes_<dataset>.nc` uses the original `system.parm7` topology, so it contains the full solvated system.

## Notes

- **Hardcoded water residue:** the angle and distance commands refer to the closest water as residue `495`. This is the residue number the water takes in the reduced topology (immediately after the solute), so it **must be updated if your solute has a different number of residues**.
- **The closest water can change between frames.** The water is re-selected every frame, so the BD angle always refers to whichever water is nearest `closest_mask` at that moment, not to a single tracked molecule.
- **Frame correspondence:** the filter data sets come from the reduced trajectory but are applied to the original trajectories. This works because both contain the same frames in the same order, so do not change `trajin` options (e.g. frame ranges or strides) in only one place.
- `dataset_name` is a bash array containing one element; `${dataset_name}` expands to that first element.
- `mkdir` will fail if `out_directory` already exists. Use `mkdir -p` to rerun in the same location.

