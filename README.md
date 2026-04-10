# gogogo-OpenSplating

# Local COLMAP + OpenSplat Pipeline (Windows)
A free and open source implementation of 3D Gaussian Splatting written in C++, focused on being portable, lean and fast. Based on OpenSplat by pierotofy.

# Scope
Under the local Windows directory d:\lzy\OpenSplat-main, starting from an existing image sequence, perform COLMAP sparse reconstruction and OpenSplat Gaussian training sequentially until a splat file is obtained.
All operations below are local – no cloud upload, no remote training, no packaging/migration.

#  Introduction & Background
3D Gaussian Splatting (3DGS) represents a breakthrough in neural rendering, offering real-time rendering capabilities while maintaining high visual quality. Unlike Neural Radiance Fields (NeRF) that rely on implicit representations (MLPs) to model volumetric radiance fields, 3DGS explicitly represents scenes using millions of 3D Gaussian primitives, enabling real-time rendering at high resolutions, efficient training compared to NeRF-based methods, explicit geometry that can be directly manipulated, and high-quality novel view synthesis.

However, NeRF is highly computation-intensive and, due to its implicit representation, poses considerable challenges in terms of editability and interactive manipulation. To overcome these limitations, 3D Gaussian Splatting has emerged as a novel paradigm for scene representation and rendering. 3DGS has achieved remarkable progress in novel view synthesis, leveraging millions of Gaussian ellipsoids for scene reconstruction and employing parallel differentiable rasterization to substantially improve rendering efficiency. For a detailed comparison, see the OpenSplat DeepWiki overview.

This project demonstrates the application of state-of-the-art neural rendering techniques for reconstructing 3D scenes from multi-view images, specifically focusing on UAV aerial imagery datasets.

# What is OpenSplat?
OpenSplat is a free and open-source implementation of 3D Gaussian Splatting written in C++, focused on being portable, lean and fast. It transforms camera poses and sparse points from various 3D reconstruction systems into optimized 3D Gaussian scene representations that can be viewed, edited, and rendered in other software.

## Core Components
Input Processing: Handles loading and parsing camera poses and sparse points from various formats (COLMAP, OpenSfM, ODM, OpenMVG, or Nerfstudio).
Gaussian Model: Each Gaussian is represented by position (3D mean), scale (size in 3 dimensions), rotation (quaternion), color features (including spherical harmonics for view-dependent effects), and opacity.
Rendering Pipeline: Projects 3D Gaussians to 2D screen space based on camera parameters, computes view-dependent colors using spherical harmonics, and rasterizes with alpha blending to produce the final image.
Adaptive Density Control: Implements densification (splitting/cloning Gaussians with high view-space gradients), pruning (removing Gaussians with low opacity), and alpha reset.

## How 3D Gaussian Splatting Works
The algorithm follows these key steps:
Initialization: A Structure-from-Motion (SfM) system like COLMAP provides initial camera poses and a sparse point cloud, which seeds the Gaussian model.
Forward Pass: Randomly select a training camera viewpoint, project 3D Gaussians to the camera plane, and render the scene.
Loss Calculation: Compute combined L1 and SSIM loss between rendered and ground truth images.
Backpropagation: Update Gaussian parameters through gradient-based optimization.
Densification & Pruning: Periodically split or duplicate Gaussians in high-gradient regions and prune low-opacity Gaussians.

## Training Parameters
OpenSplat provides various parameters to control the training process:

| Parameter | Description | Default |
|-----------|-------|-------------|
| `num-iters` | Training iterations | 30000 |
| `downscale-factor ` |	Scale input images by this factor|	1.0
| `sh-degree `|	Maximum spherical harmonics degree|	3
| `ssim-weight `|	Weight for structural similarity loss|	0.2
| `refine-every `|	Split/duplicate/prune Gaussians every N steps|	100
| `warmup-length `|	Only start densification after N steps|	500
| `densify-grad-thresh `|	Gradient threshold for Gaussian splitting|	0.0002

For detailed information about building and installing OpenSplat, see the official repository.

# Dataset Characteristics
The AMtown02 dataset used in this workflow consists of UAV (Unmanned Aerial Vehicle) imagery captured over terrain. Key characteristics include:
| Property | Value |
|----------|-------|
| **Dataset Name** | AMtown02 |
| **Number of Images** | 20 (for clip subset) to full sequence |
| **Image Format** | PNG/JPEG (frame_XXXXXX.png, six‑digit numbering) |
| **Initial SfM Points** | ~3343 |
| **Camera Model** | Pinhole |
| **Source** | UAVScenes / Aerial Imagery |

## Dataset Characteristics
Temporal Coverage: Single capture session (timestamp-based filenames)
Spatial Coverage: Terrain region suitable for aerial reconstruction
Capture Pattern: Sequential flight path (video‑like sequence)
Ground Sample Distance: UAV-typical resolution
Image Selection Criteria: To improve reconstruction stability and reduce computation, select a subset with continuous sequence, good overlap, low blur, and stable motion.

## Data Structure

```bash
data/
├── images/
│   ├── frame_000200.png
│   ├── frame_000210.png
│   └── ... (selected frames)
└── sparse/0/
    ├── cameras.bin
    ├── images.bin
    └── points3D.bin
```

# Prerequisites
Make sure the following are present on your system:
| Component	| Expected path (example) |
|----------|-------|
|COLMAP (CUDA version) |	d:\lzy\OpenSplat-main\colmap-x64-windows-cuda\bin\colmap.exe
|OpenSplat executable |	d:\lzy\OpenSplat-main\opensplat (1)\opensplat\opensplat.exe
|Helper scripts	 |  d:\lzy\OpenSplat-main\OpenSplat-main\scripts\make_clip_scene.ps1 d:\lzy\OpenSplat-main\OpenSplat-main\scripts\colmap_reconstruct.ps1

The script colmap_reconstruct.ps1 automatically finds COLMAP. Alternatively, set the environment variable COLMAP_EXE to point to your colmap.exe.


# Step 1 – Verify Program and Script Locations
Before starting, confirm that the following paths exist (if your installation paths differ, replace them in the commands later):

d:\lzy\OpenSplat-main\colmap-x64-windows-cuda\bin\colmap.exe
d:\lzy\OpenSplat-main\opensplat (1)\opensplat\opensplat.exe
d:\lzy\OpenSplat-main\OpenSplat-main\scripts\make_clip_scene.ps1
d:\lzy\OpenSplat-main\OpenSplat-main\scripts\colmap_reconstruct.ps1

COLMAP is automatically located by colmap_reconstruct.ps1; alternatively, you can set the environment variable COLMAP_EXE to point to colmap.exe.

# Step 2 – Prepare Image Data
The full dataset should be placed, for example, at:

d:\lzy\OpenSplat-main\AMtown02_scene\images\

Example image filenames: frame_000200.png (six‑digit numbering).

If you plan to run COLMAP directly on the full scene, you can skip the next step and later set the Workspace to the directory containing AMtown02_scene.

# Step 3 – (Optional) Generate a Sub‑set Workspace
If the number of frames is too large, or if disk space or time is limited, you can first copy a range of frames into an independent folder and then run COLMAP on that folder.
In PowerShell (modify the start frame, end frame, and step as needed):

```bash
powershell -ExecutionPolicy Bypass -File "d:\lzy\OpenSplat-main\OpenSplat-main\scripts\make_clip_scene.ps1" `
    -SourceWorkspace "d:\lzy\OpenSplat-main\AMtown02_scene" `
    -StartIndex 200 -EndIndex 7498 -Step 10
```

After execution, a directory similar to d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene\ will be created, under which the images\ subfolder contains the subset.
Use the -DestWorkspace parameter if you need a custom output directory.

# Step 4 – Run the Full COLMAP Pipeline
Choose a workspace directory (the full scene or the clip generated in the previous step). This directory must already have an images\ subfolder.
Execute:

```bash
powershell -ExecutionPolicy Bypass -File "d:\lzy\OpenSplat-main\OpenSplat-main\scripts\colmap_reconstruct.ps1" `
    -Workspace "d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene"
```

The script performs in order: feature extraction → sequential matcher (for video sequences) → mapper.
If you have an unordered photo set, add -Exhaustive at the end of the above command (runtime will increase significantly).
While the script is running, do not open a second COLMAP instance that accesses the same database.db.

#  Step 5 – Verify COLMAP Finished Successfully
Proceed to OpenSplat only when both of the following conditions are met:

The PowerShell script finishes without errors.
The following files exist:
<Workspace>\sparse\0\cameras.bin
<Workspace>\sparse\0\images.bin
<Workspace>\sparse\0\points3D.bin

The same directory should also contain database.db; the training phase depends on the COLMAP project and the image correspondences.

# Step 6 – Run OpenSplat Training
In PowerShell, change to the directory where you want the output splat.ply (usually the project root), then execute:

```bash
cd "d:\lzy\OpenSplat-main"
& "d:\lzy\OpenSplat-main\opensplat (1)\opensplat\opensplat.exe" `
    "d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene" -n 5000
```

-n specifies the number of iterations; you can change it to 2000–10000 depending on your GPU and time budget.
Add -o <path> if you need to specify an output file.
Run opensplat.exe --help to see other parameters.

# Step 7 – Confirm Training Completion
After training finishes normally, the current working directory (the one you cd to in the previous step) should contain splat.ply and cameras.json.
If you used -o, look for the output file at the specified path.

# Step 8 – View the Result Locally
Open the resulting splat.ply or exported .splat file with a locally installed 3D Gaussian viewer or editor
(e.g.[, common web viewers, SuperSplat, CloudCompare, etc.](https://playcanvas.com/viewer), depending on the tools you have installed).

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
