# gogogo-OpenSplating

# Local COLMAP + OpenSplat Pipeline (Windows)
This repository provides a step‑by‑step local workflow to go from a sequence of images to a final .splat file using COLMAP (sparse reconstruction) and OpenSplat (Gaussian Splatting training).
All operations run on a Windows machine – no cloud uploads, no remote training, no packaging/migration.
Based on OpenSplat by pierotofy.

# Prerequisites
Make sure the following are present on your system:

Component and Expected path (example)
COLMAP (CUDA version)	d:\lzy\OpenSplat-main\colmap-x64-windows-cuda\bin\colmap.exe
OpenSplat executable	d:\lzy\OpenSplat-main\opensplat (1)\opensplat\opensplat.exe
Helper scripts	d:\lzy\OpenSplat-main\OpenSplat-main\scripts\make_clip_scene.ps1
d:\lzy\OpenSplat-main\OpenSplat-main\scripts\colmap_reconstruct.ps1

The script colmap_reconstruct.ps1 automatically finds COLMAP.
Alternatively, set the environment variable COLMAP_EXE to point to your colmap.exe.

# Step 1 – Prepare Your Image Data
Place your full image sequence in a folder, for example:

```bash
d:\lzy\OpenSplat-main\AMtown02_scene\images\
```

Image filenames should be zero‑padded, e.g. frame_000200.png.
If you want to run COLMAP on the whole dataset, skip the next step and use the above folder as your workspace later.

# Step 2 – (Optional) Create a Sub‑set Workspace
When the full sequence is too large, you can extract a contiguous frame range into a separate workspace.

Run this PowerShell command (adjust -StartIndex, -EndIndex, -Step as needed):

```bash
powershell -ExecutionPolicy Bypass -File "d:\lzy\OpenSplat-main\OpenSplat-main\scripts\make_clip_scene.ps1" `
    -SourceWorkspace "d:\lzy\OpenSplat-main\AMtown02_scene" `
    -StartIndex 200 -EndIndex 7498 -Step 10
```

What it does:
Creates a new workspace like d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene\ containing an images\ subfolder with the selected frames.

Use -DestWorkspace to specify a custom output directory.

# Step 3 – Run Full COLMAP Reconstruction
Choose a workspace (full scene or the clip from Step 2). The workspace must already contain an images\ folder.

Execute:

```bash
powershell -ExecutionPolicy Bypass -File "d:\lzy\OpenSplat-main\OpenSplat-main\scripts\colmap_reconstruct.ps1" `
    -Workspace "d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene"
```

The script performs: feature extraction → sequential matcher (for video‑like sequences) → mapper.

For unordered photo collections, add -Exhaustive at the end (this increases runtime significantly).

Do not open another COLMAP instance that accesses the same database.db while the script is running.

# Step 4 – Verify COLMAP Success
Proceed only if both conditions are true:

The PowerShell script finishes without errors.

The following files exist inside your workspace:

```bash
<Workspace>\sparse\0\cameras.bin
<Workspace>\sparse\0\images.bin
<Workspace>\sparse\0\points3D.bin
```

The same directory should also contain database.db – it will be used by OpenSplat.

#  Step 5 – Train with OpenSplat
Navigate to the directory where you want the output splat.ply (usually the project root) and run:

```bash
cd "d:\lzy\OpenSplat-main"
& "d:\lzy\OpenSplat-main\opensplat (1)\opensplat\opensplat.exe" `
    "d:\lzy\OpenSplat-main\AMtown02_clip_000200_007498_step10_scene" -n 5000
```

Parameters:

-n : number of iterations (adjust to 2000–10000 depending on your GPU and time).
-o : optional output file path (e.g. -o my_model.splat).

Run opensplat.exe --help for all options.

# Step 6 – Confirm Training Finished
After successful training, the current working directory (the one you cd to) will contain:

splat.ply
cameras.json

If you used -o, look for the output at the specified path.

# Step 7 – View the Result Locally
Open the resulting splat.ply or .splat file with a local 3D Gaussian viewer, such as:

[SuperSplat
CloudCompare](https://playcanvas.com/viewer)

Any other web‑based or native viewer you have installed.

# Appendix – Re‑running COLMAP
If you encounter database is locked, a crash, or want to switch to a different subset:

1.Close any COLMAP GUI.

2.Delete the following inside the workspace:

database.db
*.db-wal / *.db-shm files (if present)
the entire sparse folder

3.Re‑run Step 3.

Ensure you have enough free disk space before restarting.

# License & Credits
This workflow uses:

COLMAP – Structure‑from‑Motion
OpenSplat – Real‑time Gaussian Splatting training

Adapted for local Windows usage. See each project’s repository for their respective licenses.
