
# Light validation of small molecule force field
Last updated: Mar 07 2019  
This instructions list subdirectory names for each step for easier locating of relevant files.

## Contents
Brief high-level overview of contents in this repository.
```
.
├── 01_setup
│   ├── 01_setup-dihedrals.tcl
│   ├── 02_pdbPsiInput.sh
│   ├── 03_NAMDinput.sh
│   ├── minimize.inp
│   └── psfgen
│       ├── psfgen-02
│       ├── psfgen-03
│       └── psfgen.tcl
├── 02_analysis
│   ├── 01_atom-labels
│   ├── 02_parse-diheds
│   └── view_namd_scan.tcl
├── dihed_namd
│   ├── 0
│   └── 5
├── dihed_psi4
│   ├── 0
│   │   ├── dihed-0.pdb
│   │   └── mp2-631Gd
│   └── 5
│       ├── dihed-5.pdb
│       └── mp2-631Gd
├── init_namd
├── init_psi4
├── LICENSE
├── psf_from_charmm
├── README.md
└── slurm_scripts

19 directories, 65 files
```

## I. Force field generation

1. Drew structure in MarvinSketch.
    * Saved as Tripos mol2 for CGenFF.

2. [`./cgenff`] Uploaded to [CGenFF](https://cgenff.umaryland.edu/), and downloaded resulting .str file.
    * Unchecked "Guess bond orders from connectivity"
    * Unchecked "Include parameters that are already in CGenFF"
    * Unchecked "Use CGenFF legacy v1.0"
    * Edited the name after `RESI` to `GBIN` (use the resname code you want)
    * NOTE: CGenFF website summary page says current FF version is 3.0.1 but this is NOT correct according to str header and Mackerell website.

3. [`./cgenff/`] Generate PDB file for NAMD.
    * [convertExtension.py](https://github.com/vtlim/off_psi4/blob/master/tools/convertExtension.py)
    * `python convertExtension.py -i gbi-neutral-cgenff.mol2 -o gbi-neutral-cgenff.pdb`
    * Checked out the structure in VMD. Marvin did not generate a very good starting structure.
    * Edited PDB to have resname and segname of `GBIN`.

4. [`./01_setup/psfgen/`] Generate PSF file for NAMD.
    * Command: `vmd -dispdev none gbi-neutral-cgenff.pdb -e psfgen.tcl > psfgen.out`
    * NOTE: See `psf_from_charmm` directory in this repo for an alternate approach.

5. [`./02_analysis/01_atom-labels/`] Label atom numbers in VMD. Generate image to refer to later when altering atom types or choosing atoms for dihedral scan.
    * Modify displayed label at Graphics > Labels > Properties > Format to `%a::%t`
    * Did this one by one but probably could write Tcl script if larger ligand, based on `label textformat Atoms 15 { %a::%t  }`
    * Do not render as (Tachyon, Postscript) since messes up label location. Cluster super slow, so I just screenshotted.

## II. Comparison of QM and MM minimum energy structures
NOTE: This section walks through NAMD minimization before QM minimization.
If you want QM calcns on your input structure to CGenFF, the `init_psi4/opt_mmff.py` script may be helpful. 
It uses the Quanformer package: <https://github.com/MobleyLab/quanformer>.
However, instead of installing the package, you can download the relevant modules from Github and call similar to [here](https://github.com/vtlim/off_psi4/blob/master/examples/frozen_atoms/example.py).

6. [`./init_namd/`] Minimize in NAMD. 
    * Do this prior to full scan bc starting Marvin coordinates are garbage.

7. Analyze minimized MM structure in VMD. This will be used as input for QM.
    * Load the `.psf` file and minimized `.coor` file.
    * If structure needs manual rearrangement (e.g., if not planar when you know it should be, or if you want to consider additional conformations)
        * VMD move atom: press `5`, click atom, move atom with mouse.
        * VMD move group of atoms: create new representation (`Graphics` > `Representations`) with selected atoms, press `9`, rotate/translate with mouse.
        * [VMD manual](https://www.ks.uiuc.edu/Research/vmd/current/ug/node33.html)
    * For each structure, `File` > `Save coordinates` > select `all` atoms. Save as `.pdb` or `.xyz` format.

8. [`./init_psi4/`] Geometry minimize with QM using Psi4.
    * Directly lifted coordinates from previous step for Psi4 input file. 

9. Compare minimized structures from steps 6 and 8 in VMD.
    * Based on agreement of minima structures, you may need to change atom types. In that case, remember to regenerate the psf file (step 4).
    * When I added improper terms in str file, it wasn't recognized by psfgen. So I modified the generated PSF directly.
        * In the `2 !NIMPHI: impropers` subheading, I added the atom numbers for new impropers (4 atoms/improper).
        * Also incremented subheader value next to NIMPHI by number of impropers added.
        * See example in `01_setup/psfgen/` comparing the PSF in `psfgen-02` and `psfgen-03` .
    * Also had to fix some of the atom name columns in the last column of generated PDB file bc some were missing.

## III. Comparison of QM and MM dihedral scan profiles

10. Analyze molecule to prepare for dihedral scan.
    * Identify atoms to be rotated.
        * Example: H9, N5, C8, N4, H8, H7, H6, not N3
    * Identify PDB numbers corresponding to above.
        * Example: 22 13 11 12 21 20 19
    * Determine which atoms will define dihedral angle.
        * Example: N2 C7 N3 C8 (8 9 10 11)

11. [`./01_setup/`] Generate structures for dihedral scan.
    * Subtract 1 from PDB numbers for use in Tcl script with VMD, and for NAMD
    * But use the PDB numbers as in when restraining in Psi4
    * Edit and run `01_setup-dihedrals.tcl`
        * Name of input PDB
        * Where output PDBs are written
        * Atom numbers of whole group to be rotated
        * Atom numbers of the four atoms to define dihedral. 
            * NOTE: Order matters. Atoms should be listed in order, with central atoms in middle. If [A B C D] does not work, try [D C B A].
    * Edit and run `02_pdbPsiInput.sh`
        * Main directory location
        * Name of theory subdirectory
        * Atom numbers to be frozen
        * Method and basis set used
        * Any other desired QM parameters
    * Edit and run `03_NAMDinput.sh`
        * Main directory location
        * Name of reference input file
        * Atom numbers of the dihedral for restraint

12. Run the QM and MM jobs.

13. [`./02_analysis/02_parse-diheds`] Analyze scans.
    * Get NAMD dihedral output angle from .coor files.
        * `vmdt -e ../12_analysis/get_namd_diheds.tcl -args /DFS-B/DATA/mobley/limvt/3_neutral/11_setup-namd/psfgen-03/gbin_psfgen.psf 7 8 9 10`
        * May want to sort resulting dat file (in vim: `:sort n`).
    * Compute/draw the profiles.
        * `python dihed_parser.py --qdir /path/to/qm_results/ --qfile output.dat --mdir /path/to/mm_results/ --mfile minimize.out --show --save`

14. [`./02_analysis/03_view`] View results.
    * TODO: add view scripts


## Notes to VTL

Sources of example materials:
 * PSF: `/DFS-B/DATA/mobley/limvt/3_neutral/11_setup-namd/psfgen-03/gbin_psfgen.psf`
 * PDB: `/DFS-B/DATA/mobley/limvt/3_neutral/init_namd2/03_point-out/final.pdb`
 * init NAMD input: `/DFS-B/DATA/mobley/limvt/3_neutral/init_namd2/minimize.inp`
 * init Psi4 input: `/DFS-B/DATA/mobley/limvt/3_neutral/init_psi4/01_mp2-631Gd/input.dat`
 * dihed NAMD input: `/DFS-B/DATA/mobley/limvt/3_neutral/dihed_namd`
 * dihed Psi4 input: `/DFS-B/DATA/mobley/limvt/3_neutral/dihed_psi4`

