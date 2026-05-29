# UserPDBModeling-PocketDownload

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/codeKrantz/UserPDBModeling-PocketDownload/blob/main/UserModeling%2BPocketDownload.ipynb)

A tool developed for **Chowdhury Labs at Iowa State University** that allows researchers to load enzyme structures, visualize them interactively, and automatically extract and download the binding pocket as a clean PDB file — ready for downstream docking or modeling workflows.

---

## What It Does

Computational drug discovery and protein modeling pipelines often require isolating just the **active site** (or "pocket") of a protein — the small region where a ligand or substrate binds — rather than working with the full structure. Doing this by hand in PyMOL or similar tools is repetitive and error-prone.

This notebook automates that workflow entirely inside Google Colab:

1. **Load a protein structure** — either by entering a public PDB ID (fetched live from the RCSB database) or by uploading your own `.pdb` file from disk.
2. **Visualize the full structure** interactively using MolView, rendered directly in the notebook.
3. **Automatically detect the active site** by finding all protein residues within a configurable radius (default: 5.0 Å) of any bound heteroatom ligand.
4. **Extract and save the pocket** as a standalone `.pdb` file, named by structure and ligand.
5. **Download the pocket file** with a single button click, ready for use in docking software like AutoDock, Vina, Glide, or similar tools.

---

## How It Works

### Input Modes

The notebook supports two input methods, toggled via a widget in the interface:

**PDB ID mode** — the user types a 4-character RCSB PDB identifier (e.g., `1ABC`). The notebook validates that the ID exists by making a live HTTP request to `https://files.rcsb.org/download/{ID}.pdb`, downloads the file to the Colab runtime, and proceeds to processing.

**Upload mode** — the user uploads a local `.pdb` file directly into Colab using the `google.colab.files` uploader. The file is saved to a stable path (`/content/current_upload.pdb`) so it persists across widget interactions within the session.

### Active Site Detection

The core scientific logic uses **BioPython's** `PDBParser` to parse the structure into a hierarchy of models, chains, residues, and atoms. The detection algorithm works as follows:

1. All **polymer residues** (standard amino acids — residues where the hetero flag is a blank space) are collected and their atomic coordinates stored as a NumPy array.
2. All **heteroatom residues** are identified — these are non-polymer, non-water small molecules (ligands, cofactors, metals) indicated by a non-blank, non-`W` hetero flag in the PDB residue ID.
3. For each heteroatom, the Euclidean distance from that atom to every protein atom is computed vectorially using NumPy — `np.sum((prot_coords - h_xyz) ** 2, axis=1)` — and any protein atom within the squared radius cutoff is flagged.
4. The **unique set of protein residues** containing at least one flagged atom is collected — these are the active-site residues.
5. A contacts DataFrame is also generated, recording which protein residues contact which ligand residues, for traceability.

The radius is set by the `ACTIVE_SITE_RADIUS_ANGSTROM` constant (default `5.0` Å), which can be adjusted at the top of the cell.

### Pocket Extraction and Saving

BioPython's `PDBIO` and a custom `Select` subclass (`SelectResidues`) are used to write only the active-site residues to a new `.pdb` file. The output filename follows the pattern `pockets/{structure_label}_{ligand_resname}.pdb`. The `os.makedirs` call ensures the `pockets/` directory is created if it doesn't exist.

### Visualization

Both the full structure and the extracted pocket are visualized using **MolView**, an interactive 3D molecular viewer that renders directly in the Colab output cell. The viewer is initialized with `mv.view()`, the PDB text is loaded with `v.addModel()`, and custom coloring is applied before `v.show()` renders it.

### UI Layer

The interactive interface is built entirely with **ipywidgets**:

- A `ToggleButtons` widget switches between PDB ID and upload modes.
- A `Text` widget accepts the PDB ID.
- `Button` widgets trigger upload, run, and download actions.
- An `Output` widget captures all status messages and errors, cleared on each new run.
- The **Download Pocket** button is disabled by default and only becomes active once a pocket file has been successfully written to disk.

---

## Repository Structure

```
UserPDBModeling-PocketDownload/
│
├── UserModeling+PocketDownload.ipynb   # Main notebook — all logic and UI
└── README.md                           # This file
```

Pocket files generated during a session are saved to a `pockets/` directory created at runtime inside the Colab environment.

---

## Setup and Usage

### Requirements

All dependencies are installed automatically in the first notebook cell. A Google Colab environment is assumed.

```
biopython
nglview
molview
pandas
requests
ipywidgets
```

> **Note:** The InterProScan setup step (`git clone` + `pip install -r requirements.txt`) may require a runtime restart after the first install. Dismiss any third-party widget popups and re-run the setup cell if needed.

### Running the Notebook

1. Open the notebook in Google Colab using the badge at the top of this README.
2. Run the **Setup and Dependencies** cell first. Restart the runtime if prompted.
3. Run the **Interactive Interface** cell. The UI will appear below the code.
4. Choose your input mode (PDB ID or Upload File).
5. Enter a PDB ID or upload a `.pdb` file, then click **Run**.
6. Once processing completes, click **Download Pocket** to save the extracted pocket file.
7. Optionally, run the **Visualize Last Generated Active Site Pocket** cell to inspect the pocket in 3D.

### Configuration

At the top of the main interface cell, two constants control behavior:

```python
ACTIVE_SITE_RADIUS_ANGSTROM = 5.0   # Distance cutoff for pocket detection
EXCLUDE_WATER = True                 # Whether to ignore water molecules (HOH)
```

Adjust `ACTIVE_SITE_RADIUS_ANGSTROM` to capture a larger or smaller shell of residues around the ligand. A value of 4–6 Å is standard for active-site definition in most docking prep workflows.

---

## Example Output

For a structure like `1HSG` (HIV-1 protease with an inhibitor), the notebook will:

- Download the PDB file from RCSB
- Detect the bound ligand (e.g., `MK1`)
- Identify all protein residues within 5 Å of the ligand
- Save `pockets/1HSG_MK1.pdb`
- Enable the Download button

The status log will report something like:

```
Processing public PDB ID: 1HSG...
[SAVED] /content/1HSG.pdb
[READY] Visualizing 1HSG...
[SAVED] ./pockets/1HSG_MK1.pdb with 24 residues
[READY] Pocket file saved at: ./pockets/1HSG_MK1.pdb
[INFO] Active-site residues found: 24
[INFO] Unique ligand residues involved: 1
```

---

## What I Learned

This project was built as part of research support work for Chowdhury Labs and represented my first time working at the intersection of Python and structural biology. Key skills and concepts developed:

**Structural biology data formats** — understanding the PDB file format, how RCSB organizes protein structure data, and the meaning of hetero flags, chain identifiers, residue sequence numbers, and insertion codes in the BioPython residue ID tuple `(hetflag, resseq, icode)`.

**BioPython's PDB module** — using `PDBParser`, navigating the structure hierarchy (Structure → Model → Chain → Residue → Atom), implementing custom `Select` subclasses to filter which residues get written by `PDBIO`, and understanding the difference between polymer residues and heteroatoms.

**Vectorized spatial computing with NumPy** — replacing slow nested loops for distance calculation with a vectorized approach: stacking all protein atom coordinates into a matrix and computing squared distances to each ligand atom in a single array operation. This is a meaningful performance improvement when a protein has tens of thousands of atoms.

**Pandas for structured biological data** — organizing active-site residues and ligand contacts into DataFrames with meaningful columns, which makes the output inspectable and extensible (e.g., you could export the contacts table for further analysis).

**Google Colab-specific patterns** — using `google.colab.files` for upload and download within a notebook, managing file paths in the ephemeral Colab runtime, handling widget state across multiple cell executions, and writing UI with ipywidgets, including button callbacks, output capture with `clear_output()`, and conditional widget visibility.

**Software design for scientific tools** — separating concerns between data fetching, structure parsing, active-site logic, file I/O, and UI; using a state dictionary (`latest_output_file`, `uploaded_pdb_file`) to pass data between widget callbacks without global mutation; and writing defensive error handling so the UI reports meaningful messages rather than crashing silently.

---

## Potential Extensions

- Support for `.cif` / mmCIF files (using BioPython's `MMCIFParser`, which is already imported)
- Second-shell residue extraction (residues within N Å of the first-shell pocket)
- Batch processing of multiple PDB IDs from a list or CSV
- Automatic pocket volume estimation
- Integration with AutoDock Vina for one-click docking from within the notebook

---

## Acknowledgments

Developed for and in collaboration with **Chowdhury Labs, Iowa State University**.
Uses [BioPython](https://biopython.org/), [MolView](https://molview.org/), [nglview](https://nglviewer.org/nglview/latest/), and the [RCSB Protein Data Bank](https://www.rcsb.org/).
