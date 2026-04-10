# gogogo-OpenSplating

# A Local Pipeline with OpenSplat

# Executive summary
We reconstruct an AMtown02 urban scene from UAV-style sequential frames. A decimated subset (frames 200–7498, every 10th frame, ~730 images) is processed with COLMAP, then OpenSplat optimizes 3D Gaussians for real-time-style novel view synthesis.

# Background
3D Gaussian Splatting (3DGS) represents a scene with millions of anisotropic 3D Gaussians optimized via differentiable rasterization. Compared with NeRF, it often enables faster training and real-time rendering with explicit geometry.

OpenSplat (C++/LibTorch) ingests standard COLMAP projects (sparse points + cameras) and outputs splat.ply / cameras.json (or custom names). COLMAP provides camera poses and sparse initialization.

Densification & Pruning: Periodically split or duplicate Gaussians in high-gradient regions and prune low-opacity Gaussians.

# Objectives
Build a reproducible Windows pipeline from raw frames to splat output.

Handle large sequential subsets (disk, SQLite locks, long Mapper).

Document commands, paths, and deliverable metrics for coursework.


# Dataset(AMtown02)

| Property | Value |
|----------|-------|
| **Dataset Name** | AMtown02 |
| **Frame range** | 200 – 7498 |
| **Decimation** | Step 10 (one frame every 10)|
| **Approx. copied images** | ~730 (confirm via make_clip_scene) |
| **Image files** | PNG, frame_XXXXXX.png |
| **COLMAP camera model (typical)** | SIMPLE_RADIAL, single_camera |
| **Resolution (typical from COLMAP logs)** | ≈ 2448 × 2048 |

## Data layout

```bash
d:\lzy\OpenSplat-main\
├── AMtown02_scene\images\
└── AMtown02_clip_000200_007498_step10_scene\
    ├── images\
    ├── database.db
    └── sparse\0\
        ├── cameras.bin
        ├── images.bin
        └── points3D.bin
```

# Methodology
Pipeline:

(1) copy subset with make_clip_scene.ps1;

(2) COLMAP feature_extractor → sequential_matcher (overlap=25) → mapper; 

(3) OpenSplat trains Gaussians with photometric loss and adaptive densification/pruning (see opensplat --help).


#  system configuration
| Component	| Specification|
|----------|-------|
|OS|	Windows 11|
|COLMAP|	Windows CUDA build (colmap-x64-windows-cuda)|
|OpenSplat|	Pre-built opensplat.exe, CUDA|
|Scripts|	d:\lzy\OpenSplat-main\OpenSplat-main\scripts\|


# Prerequisites
Make sure the following are present on your system:
| Component	| Expected path (example) |
|----------|-------|
|COLMAP (CUDA version) |	d:\lzy\OpenSplat-main\colmap-x64-windows-cuda\bin\colmap.exe
|OpenSplat executable |	d:\lzy\OpenSplat-main\opensplat (1)\opensplat\opensplat.exe
|Helper scripts	 |  d:\lzy\OpenSplat-main\OpenSplat-main\scripts\make_clip_scene.ps1 d:\lzy\OpenSplat-main\OpenSplat-main\scripts\colmap_reconstruct.ps1

The script colmap_reconstruct.ps1 automatically finds COLMAP. Alternatively, set the environment variable COLMAP_EXE to point to your colmap.exe.


# Step 1 – Prepare Image Data
The full dataset should be placed, for example, at:

d:\lzy\OpenSplat-main\AMtown02_scene\images\

Example image filenames: frame_000200.png (six‑digit numbering).

# Step 2 – (Optional) Generate a Sub‑set Workspace
If the number of frames is too large, or if disk space or time is limited, you can first copy a range of frames into an independent folder and then run COLMAP on that folder.
In PowerShell (modify the start frame, end frame, and step as needed):

```bash
powershell -ExecutionPolicy Bypass -File "d:\lzy\OpenSplat-main\OpenSplat-main\scripts\make_clip_scene.ps1" `
    -SourceWorkspace "d:\lzy\OpenSplat-main\AMtown02_scene" `
    -StartIndex 200 -EndIndex 7498 
```

After execution, a directory similar to d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene\ will be created, under which the images\ subfolder contains the subset.
Use the -DestWorkspace parameter if the program need a custom output directory.

# Step 3 – Run the Full COLMAP Pipeline
Choose a workspace directory (the full scene or the clip generated in the previous step). This directory must already have an images\ subfolder.
Execute:

```bash
powershell -ExecutionPolicy Bypass -File "d:\lzy\OpenSplat-main\OpenSplat-main\scripts\colmap_reconstruct.ps1" `
    -Workspace "d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene"
```

The script performs in order: feature extraction → sequential matcher (for video sequences) → mapper.
If you have an unordered photo set, add -Exhaustive at the end of the above command (runtime will increase significantly).
While the script is running, do not open a second COLMAP instance that accesses the same database.db.

#  Step 4 – Verify COLMAP Finished Successfully
Proceed to OpenSplat only when both of the following conditions are met:

The PowerShell script finishes without errors.
The following files exist:
<Workspace>\sparse\0\cameras.bin
<Workspace>\sparse\0\images.bin
<Workspace>\sparse\0\points3D.bin

The same directory should also contain database.db; the training phase depends on the COLMAP project and the image correspondences.

# Step 5 – Run OpenSplat Training
In PowerShell, change to the directory where you want the output splat.ply (usually the project root), then execute:

```bash
cd "d:\lzy\OpenSplat-main"
& "d:\lzy\OpenSplat-main\opensplat (1)\opensplat\opensplat.exe" `
    "d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene" -n 5000
```

-n specifies the number of iterations; you can change it to 2000–10000 depending on your GPU and time budget.
Add -o <path> if you need to specify an output file.
Run opensplat.exe --help to see other parameters.

# Step 6 – Confirm Training Completion
After training finishes normally, the current working directory (the one you cd to in the previous step) should contain splat.ply and cameras.json.
If you used -o, look for the output file at the specified path.

# Step 7 – View the Result Locally
Open the resulting splat.ply or exported .splat file with a locally installed 3D Gaussian viewer or editor
(e.g.[, common web viewers, SuperSplat, CloudCompare, etc.](https://playcanvas.com/viewer), depending on the tools you have installed).

# Results — deliverable files (measured)
Source files: zcameras(3).json, zsplat(3).ply

<img width="1034" height="1145" alt="bdc21156582e71f7302e0cebbab0d8ed" src="https://github.com/user-attachments/assets/a4410c5f-d521-4478-bded-aefe29d6f87c" />


| Metric	| Value |
|----------|-------|
|Cameras in JSON|	721|
|Image size (first camera)|	2447 × 2047 px|
|Gaussians (PLY vertex count)|	2,109,944|
|zcameras file size|	279,991 bytes|
|zsplat file size|	523,267,675 bytes (499.0 MB)|
|PLY format|	format binary_little_endian 1.0|
|SH coefficients (header)|	3 DC + 45 f_rest|

# Training log statistics

Parsed from: opensplat_step5_bg.log. If the workspace in the log does not match the subset that produced your zsplat file, treat loss numbers as a separate experiment.

=== OpenSplat background start 2026-04-10T09:23:53.4651656+08:00 workspace=d:\lzy\OpenSplat-main\AMtown02_clip_000200_004200_step5_scene -n 5000 ===


| Item	| Value |
|----------|-------|
|COLMAP points at train start|	676573
|Logged step lines|	500
|First step / loss|	10 / 0.227556
|Last step / loss|	5000 / 0.100927
|Min / max loss|	0.027898 / 0.278602
|Mean loss|	0.128651
|First→last reduction|	55.65%

# Issues encountered & mitigations
SQLite database is locked: stop other COLMAP users; delete db + wal + shm + sparse; rerun.

PowerShell Stop on COLMAP stderr: colmap_reconstruct.ps1 uses Continue + exit-code checks.

Log file lock during background runs: use FileStream append with FileShare.Read and retries.

Disk space: delete old clip workspaces and huge database.db when switching subsets.

# Strengths & future work

Strengths: native Windows + CUDA; sequential matcher suited to flight lines; scripted subset and COLMAP; background logging.

PSNR/SSIM on held-out views; train/test split.

Archive a log for every run that matches exported PLY/cameras.

Hyperparameter sweep; compare iteration count vs. quality.

Optional .splat export and viewer-side cleanup.

# References
Kerbl et al., 3D Gaussian Splatting for Real-Time Radiance Field Rendering, SIGGRAPH 2023.

Schönberger & Frahm, Structure-from-Motion Revisited, CVPR 2016.

COLMAP: https://colmap.github.io/

OpenSplat: https://github.com/pierotofy/OpenSplat


# Appendix – When You Need to Re‑run COLMAP
If you encounter a database is locked error, a mid‑way failure, or want to switch to a different subset:

Close COLMAP.

Delete the following inside the workspace:

database.db

any *.db-wal / *.db-shm files

the entire sparse folder

Then re‑run Step 4.

Make sure you have enough disk space.

# License & Credits
This workflow uses:

COLMAP – Structure‑from‑Motion
OpenSplat – Real‑time Gaussian Splatting training

Adapted for local Windows usage. See each project’s repository for their respective licenses.
