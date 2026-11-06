# gundam_lbl_config

GUNDAM configuration, fit scripts and analysis macros for the HK fake-data
oscillation sensitivity. The generated inputs (selected samples, covariances,
oscillation grid) live in `$data_dir`, produced by the `mock_lbl_inputs`
repository; the fit results stay here under `outputs/`.

    config.yaml       top-level fit config, Asimov data set
    inputs/           everything config.yaml includes, all paths relative to this directory
    asimov.sh         the plain fit, everything free
    grid.sh           3D GridSearch over sin^2(theta23), delta-CP, dm^2_32
    grid1d.sh         1D GridSearch in each of the three
    profile_*.sh      profile likelihood in each of the three, one fit per point
    profileGrid.C     1D and 2D profiles of any GridSearch output
    compare3way.C     profiles vs 1D grid vs 3D grid, one canvas per parameter
    plotCovariance.C  the prior covariance, correlation and sigmas
    outputs/          fit results and SLURM logs, created as needed

## Before running

    export data_dir=/path/to/generated/data      # fakeselect_*.root, parameters/
    source <gundam>/setup.sh
    source <gundamOscAnaTools>/.../setup.NuOscillator.sh
    cd gundam_lbl_config/

`data_dir` is read by the configs as `${data_dir}`, which GUNDAM expands.
`NUOSCILLATOR_ROOT` is set by the NuOscillator setup script. The SLURM scripts
source `$HOME/setup_neut.sh` themselves. Everything is run from this directory:
the configs use relative paths and the `#SBATCH --output` lines point at
`outputs/logs/`.

## Fits

    sbatch asimov.sh          1 job              outputs/asimov/
    sbatch grid.sh            51 array tasks     outputs/grid/grid_<task>.root
    sbatch grid1d.sh          3 array tasks      outputs/grid1d/grid1d_{dcp,sin23,dm32}.root
    sbatch profile_dcp.sh     48 array tasks     outputs/profile/
    sbatch profile_dm32.sh    50 array tasks     outputs/profile/
    sbatch profile_sin23.sh   50 array tasks     outputs/profile/

The grids are defined in `inputs/fitter/configGridSearch.yaml` and
`configGrid1D_*.yaml`; the `use*.yaml` files next to them are the `-of`
overrides that swap the minimizer. `grid.sh` splits the 3D grid into 51 blocks
of 1968 nodes with `firstNode`/`nNodes`. The profile points are the 148 files
in `inputs/profile/`, described in the README there.

## GENIE inputs

To fit with the GENIE cross-section parameters instead of NEUT, set `data_dir`
to a data_dir made by `mock_lbl_inputs/scripts/genie` and add the override:

    gundamFitter -c config.yaml -of inputs/parameters/useGENIE.yaml

The SLURM scripts are unchanged, so add `-of inputs/parameters/useGENIE.yaml`
to their gundamFitter line by hand if needed.

## Analysis

    root -l -b -q 'profileGrid.C("outputs/grid/grid_*.root","outputs/grid_profiles.root")'
    root -l -b -q 'compare3way.C("outputs/profile","outputs/grid1d","outputs/grid","outputs/compare3way.root")'
    root -l -b -q "plotCovariance.C(\"$data_dir/parameters\",\"outputs/priorCovariance.root\")"

All three write ROOT files holding TCanvas objects. Contours and curves are
labelled by raw Delta(chi2); no sigma or confidence-level conversion is drawn,
since that needs Feldman-Cousins.
