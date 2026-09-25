---
name: pdb-viewer-builder
description: Build and launch a self-contained interactive PDB structure viewer in a user-specified output folder, preserving existing work and verifying the browser result.
---

# Build and launch a PDB viewer

Create an interactive browser viewer for a supplied PDB structure, save the
complete, reproducible app in the requested output directory, and launch it
for inspection. This skill is for building or rebuilding a viewer; it is not a
substitute for a structure-analysis or database-query skill.

## Inputs and scope

Before editing, identify:

- the PDB ID or local PDB/mmCIF input;
- the exact output directory;
- whether the viewer must work offline or may use an explicitly approved
  network dependency;
- the requested scientific or explanatory focus.

If the structure input or output directory is ambiguous, ask one focused
question. Do not substitute another structure without saying so.

## Preserve existing work

Inspect the output directory before writing anything. Never overwrite an
existing app, data file, or unrelated run. Prefer this order:

1. reuse a complete existing viewer when it matches the requested input;
2. create only missing files when the existing app can be safely extended;
3. create a new run subdirectory if the existing contents are incompatible.

If a required file would need to be replaced, show the path and ask for
confirmation. Keep source coordinates and provenance intact.

## Build a useful first version

Create a minimal browser app before adding elaborate analysis. Use real
coordinates from the supplied input and a documented molecular rendering
library. Prefer a vendored runtime or another already-available local
dependency; do not install packages or add a network dependency without
approval.

A successful first version has:

- `index.html` with a visible viewer mount and loading/error states;
- a local or explicitly disclosed rendering dependency;
- actual coordinate data, with provenance and source URL or input path;
- working rotate, zoom, and reset controls;
- responsive styling and readable labels;
- a short README explaining how to open the app and what it displays.

Then add controls that are supported by the data, such as chain visibility,
cartoon/surface/atomic representations, ligand or heme display, selection,
and camera landmarks. Derive chains, residues, ligands, and available models
from the parsed structure; do not hardcode identifiers from a different PDB
entry. Chain visibility must also affect associated ligands, labels, and
selection highlights.

Keep interpretation separate from observation. Label deposited metadata as
such, identify the model or assembly being shown, and do not claim simulated
behavior, predicted confidence, or molecular interactions that the data does
not establish. If no ligand is present, say so instead of inventing one.

## Launch the app

After the files are present:

1. choose a free loopback port;
2. start a static server from the output directory, for example:

   ```bash
   python3 -m http.server 8765 --bind 127.0.0.1
   ```

3. open `http://127.0.0.1:<port>/` with the browser tool;
4. inspect the page, loading/error state, canvas, controls, and a few data
   labels.

Use the browser inspection tool when available. Do not claim that the app was
launched or visually verified merely because files were written or a server
command returned successfully. If browser inspection is unavailable, say
exactly what was checked and ask the user to inspect the remaining behavior.

Keep the server running after launch unless the user asks to stop it. Report
the URL, port, files created or reused, and the checks performed.

## Handover format

Return:

- the viewer title and source structure;
- the output directory;
- the launch URL and whether it was opened;
- the files created, reused, or deliberately left unchanged;
- the controls and metadata verified;
- untested behavior and scientific limitations.

Do not call the result “generated” and “validated” interchangeably. Keep
those claims separate.
