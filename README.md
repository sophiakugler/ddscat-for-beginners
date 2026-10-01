# DDSCAT for Beginners

This repository contains a small collection of notes and Python scripts for setting up, running, and
analysing DDSCAT calculations. It was created during a 2-month summer internship at
Christian-Albrechts-Universität zu Kiel and is intended as a practical
starting point for users who are new to DDSCAT.

> **Note:** Some of the Python and shell code in this repository was
> developed with assistance from large language models, including ChatGPT
> (OpenAI). The scripts were reviewed, modified, and tested on the calculations
> used during this project. Paths and numerical parameters should still be
> checked before using them for a new setup.

## Overview

This Readme should provide all information necessary to use this repository for setting up a first successful DDSCAT-run.

- [Documentation](#1-documentation)
- [Purpose](#2-what-is-the-purpose-of-this-repository)
- [Installing DDSCAT](#3-installing-ddscat)
- [Python requirements](#4-python-requirements)
- [DDSCAT run workflow](#5-ddscat-run-workflow)
- [Additional scripts](#6-additional-analysis-scripts)
- [Plotting choices](#7-plotting-choices)
- [Minimal workflow](#8-minimal-workflow)
- [Quickstart](#9-quickstart--setting-up-a-first-ddscat-run)
- [References](#10-references-and-further-information)

## 1. Documentation

A more detailed introduction is available here:

**[DDSCAT for beginners (PDF)](ddscat_for_beginners.pdf)**

The manual explains the main DDSCAT input files, `ddscat.par`, material files,
custom `shape.dat` targets, important output files, convergence checks, and a
complete example calculation.

## 2. What is the purpose of this repository?

The workflow provided here is intended for people who never used DDSCAT before. It contains everything needed for setting up, running and analysing basic DDSCAT calculations.

### 2.1 What is DDSCAT?

DDSCAT (Discrete Dipole SCATtering) is a Fortran-code which calculates how light interacts with a particle using the Discrete Dipole Approximation (DDA). Very roughly, DDSCAT replaces a real dust grain by small dipoles and solves interaction of light with these dipoles, from which it then computes absorption and scattering. Since it needs to solve the interaction of each dipole, increasing the number of dipoles becomes more and more expensive. The mathematical details of DDA are however not needed for a first application of this tool.

For first-time users, it is recommended to directly go to the [quickstart guide](#9-quickstart--setting-up-a-first-ddscat-run).

## 3. Installing DDSCAT

DDSCAT itself is not included in this repository.

Download DDSCAT 7.3.4 from the official DDSCAT download page:

**https://ddscat.wikidot.com/downloads**

Download both:

1. **DDSCAT 7.3.4 FORTRAN code**
2. **DDSCAT 7.3.4 Examples**

The example package is needed for the `examples_exp` directory used throughout
this repository.

A convenient directory structure is:

```text
DDA/
├── src/
│   └── ddscat
├── examples_exp/
├── diel/
└── doc/
```

The exact location does not matter, but the paths in `input.toml` must be
changed accordingly.

### 3.1 Compile DDSCAT

DDSCAT is written in Fortran. On Linux, `gfortran` can be used.

```bash
gfortran --version
```

Compile DDSCAT from the source directory:

```bash
cd ~/DDA/src
make ddscat
```

The resulting executable should be located at something like:

```text
~/DDA/src/ddscat
```

Check that it exists with:

```bash
ls -l ~/DDA/src/ddscat
```

After compiling is done, this executable is the file which DDSCAT uses for running and calculating.

## 4. Python requirements

The helper scripts use Python 3. Some analysis scripts additionally require
NumPy, Matplotlib, and SciPy.

```bash
python3 -m pip install numpy matplotlib scipy
```

The automated workflow uses Python 3.11 or newer.

---

## 5. DDSCAT run workflow

This repository is meant to help users get familiar with DDSCAT without
having to understand the Fortran source code itself.

The main workflow is:

```text
input.toml
    ↓
generate_ddscat.py
    ↓
ddscat.par
    ↓
main.sh
    ↓
DDSCAT
    ↓
qtable / .avg / .sca / log files
```

The user normally only edits `input.toml`. The Python helper then generates
`ddscat.par`, and `main.sh` starts DDSCAT.

For background information on the individual DDSCAT parameters, see the
accompanying manual and the official DDSCAT User Guide.

### 5.1 Files in this repository

```text
.
├── README.md
├── input.toml
├── generate_ddscat.py
├── main.sh
├── check_run.py
├── plot_qtable.py
├── main.txt
├── ddscat_for_beginners.pdf
└── scripts/
    ├── compare_qtables.py
    ├── generate_oblate_shape.py
    ├── generate_porous_sphere.py
    ├── generate_two_material_sphere.py
    ├── plot_multiple_qtables.py
    ├── plot_shape_slice.py
    ├── runtime_vs_wavelength.py
    ├── visualize_shape_3d.py
    └── visualize_shape_slice.py

```

The workflow uses one user-editable input file:

- `input.toml`: paths, target, wavelength grid, effective radius and numerical settings

The main scripts are:

- `generate_ddscat.py`: creates the run directory and `ddscat.par`
- `main.sh`: prepares the input files and runs DDSCAT; it can also be submitted with Slurm
- `check_run.py`: checks how many wavelength/radius combinations finished and inspects the DDSCAT log
- `plot_qtable.py`: makes a first overview plot from `qtable`

The additional scripts in `scripts/` are optional examples used during the
project. They are **not required to run DDSCAT**. They can be adapted for
creating custom targets, visualising `shape.dat`, comparing calculations, and
analysing runtimes.

---

### 5.2 Before running

DDSCAT must already be compiled.

Set the path to the executable in `input.toml`:

```toml
ddscat_executable = "/star/home/USER/DDA/src/ddscat"
```

A material file is always required:

```toml
material_file = "/star/home/USER/DDA/diel/astrosil"
```

For a custom target using `FROM_FILE`, a geometry file is also required:

```toml
shape = "FROM_FILE"
shape_file = "/star/data/USER/DDSCAT/shapes/shape.dat"
```

The helper copies the material into the run directory as `diel.dat` and the
custom geometry as `shape.dat`. `ddscat.par` therefore contains short local
file names. This also avoids problems with long material paths in DDSCAT.

For targets containing more than one material, DDSCAT needs one dielectric
file per material and `NCOMP` must match the number of materials. The material
numbers stored in `shape.dat` (`ICOMPX`, `ICOMPY`, `ICOMPZ`) refer to the order
of these dielectric files. See `scripts/generate_two_material_sphere.py` for a
simple example.

For a built-in ellipsoid target, use:

```toml
shape = "ELLIPSOID"
shape_parameters = [70.0, 70.0, 70.0]
```

For `ELLIPSOID`, the three numbers describe the target dimensions in units of
the dipole spacing `d`. Here, `d` is the distance between neighbouring dipoles
on the DDSCAT lattice and `D` is the particle diameter. The ratio `D/d`
therefore tells us approximately how many dipole spacings fit across the
particle.

For

```toml
shape_parameters = [70.0, 70.0, 70.0]
```

all three dimensions are equal, so the target is a sphere with approximately
`D/d = 70`. The value `70` is **not** a physical radius in micrometres. The
physical grain size is set separately by the effective radius.

Increasing the values, for example from `[50, 50, 50]` to `[70, 70, 70]`,
represents the same physical grain with more dipoles and therefore a finer DDA
resolution, but also increases the computational cost.

Different values produce a non-spherical ellipsoid. For example,

```toml
shape_parameters = [70.0, 70.0, 35.0]
```

gives an oblate grain that is flattened along the third axis.

The values are passed directly to DDSCAT as `SHPAR1`, `SHPAR2`, and `SHPAR3`;
consult the DDSCAT manual before changing them substantially.

---

### 1. Edit `input.toml`

The most commonly changed entries are:

```toml
[paths]

# Change these paths for your system.
ddscat_executable = "/star/home/USER/DDA/src/ddscat"
run_directory = "/star/data/USER/DDSCAT/example_run"
material_file = "/star/home/USER/DDA/diel/astrosil"

[target]

# Use "FROM_FILE" for a custom shape.dat target.
# Use "ELLIPSOID" for a built-in ellipsoid/sphere.
shape = "FROM_FILE"

# Only needed for FROM_FILE.
shape_file = "/star/data/USER/DDSCAT/shapes/shape.dat"

# For ELLIPSOID, use for example:
# shape_parameters = [70.0, 70.0, 70.0]
#
# d = distance between neighbouring dipoles
# D = particle diameter
# [70,70,70] therefore corresponds to roughly D/d = 70 in each direction.
# The physical particle size is set separately by effective_radius.

[effective_radius]

# Grain radius/radius range in micrometres.
minimum_um = 1.0
maximum_um = 1.0
count = 1

# LIN = linear spacing
# LOG = logarithmic spacing
# INV = inverse spacing
spacing = "LIN"

[wavelength]

# Wavelength range in micrometres.
minimum_um = 0.3
maximum_um = 100.0
count = 60

# LOG is useful for wavelength ranges spanning several orders of magnitude.
# Change to LIN for equally spaced wavelengths.
spacing = "LOG"
```

`LIN`, `LOG`, and `INV` are accepted for the wavelength and effective-radius
grids.

The numerical settings normally do not need to be changed for a first run:

```toml
[numerics]

# Iterative solver convergence tolerance.
tolerance = 1.0e-5

# Maximum number of solver iterations.
max_iterations = 1000

solver = "PBCGS2"
fft = "GPFAFT"
polarizability = "GKDLDR"
```

---

### 2. Generate the DDSCAT input files

From the repository directory:

```bash
python3 generate_ddscat.py input.toml
```

For a `FROM_FILE` target, the run directory should then contain:

```text
ddscat.par
shape.dat
diel.dat
```

For an `ELLIPSOID` target, `shape.dat` is not needed because DDSCAT generates
the target internally.

It is a good idea to inspect the generated parameter file before the first
run:

```bash
cat /path/to/run_directory/ddscat.par
```

---

### 3. Run DDSCAT locally

The same shell script can be run directly without Slurm:

```bash
bash main.sh input.toml
```

The script first regenerates the input files and then starts the DDSCAT
executable inside the configured run directory.

A successful calculation should eventually report:

```text
DDSCAT normal termination
```

---

### 4. Run DDSCAT with Slurm

The Slurm settings are at the top of `main.sh`:

```bash
#SBATCH --job-name=ddscat
#SBATCH --partition=student-l
#SBATCH --qos=long
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=1
#SBATCH --time=1-00:00:00
#SBATCH --mem=8G
```

Adjust these lines for the local cluster if necessary.

A serial DDSCAT executable should normally request one CPU. Requesting more
CPUs does not make a serial build faster.

Submit the run with:

```bash
sbatch main.sh input.toml
```

Check the queue with:

```bash
squeue -u $USER
```

The Slurm standard output and error files are named approximately:

```text
ddscat_<jobid>.out
ddscat_<jobid>.err
```

---

### 5. Check whether the calculation finished

Run:

```bash
python3 check_run.py input.toml
```

For a calculation with 60 wavelengths and one effective radius, a complete
`qtable` should report something like:

```text
qtable rows: 60 / 60
status: DDSCAT normal termination
```

If DDSCAT failed, the script prints the final lines of the log.

A common convergence error is:

```text
FATAL ERROR IN PROCEDURE: ZBCG2WP
ITERN>ITERMX
```

This means that the iterative solver reached `max_iterations` before reaching
the requested tolerance.

Do not automatically increase the iteration limit. First check whether the
selected wavelength, material and dipole resolution are reasonable.

---

### 6. Important output files

DDSCAT can create many output files. The most useful ones for a first analysis
are:

- `qtable` — main wavelength-dependent quantities such as `Qext`, `Qabs`, `Qsca`, and `g`
- `qtable2` — additional wavelength-dependent quantities
- `wXXXrXXX.avg` — detailed averaged result for one wavelength and effective radius
- `wXXXrXXXkXXX.sca` — angular scattering and Mueller-matrix information when requested
- `target.out` — information about the target geometry
- `ddscat.log_000` — numerical log, timing information, and solver convergence

The main efficiency factors satisfy:

```text
Qext = Qabs + Qsca
```

DDSCAT defines an efficiency from its corresponding cross section using:

```text
Q = C / (pi * a_eff^2)
```

For porous targets, remember that `a_eff` is the equal-material-volume radius.
It can differ from the outer physical radius of the grain.

---

### 7. Make a quick plot

After a successful run:

```bash
python3 plot_qtable.py input.toml
```

This creates:

```text
qtable_overview.png
```

inside the run directory and plots `Qabs`, `Qsca`, and `Qext` as functions of
wavelength.

---

## 6. Additional analysis scripts

The `scripts/` directory contains small scripts used during the project.

### `compare_qtables.py`

Compares the absorption efficiency from two DDSCAT runs and plots

```text
Qabs / Qabs_reference
```

Useful for comparing porosity, shape, composition, or resolution.

Change the paths near the beginning of the file:

```python
REFERENCE = Path("run_reference/qtable")
COMPARISON = Path("run_other/qtable")
```

### `plot_multiple_qtables.py`

Plots `Qabs(lambda)` from several runs in one figure.

Edit the `RUNS` dictionary at the beginning of the file to add or remove
calculations.

### `plot_shape_slice.py`

Plots a central slice through a custom `shape.dat`.

Change:

```python
SHAPE_FILE = Path("shape.dat")
SLICE_Z = 0
```

The material number stored in `ICOMPX` is used to distinguish different
materials.

### `generate_porous_sphere.py`

Generates custom porous spherical targets as `shape.dat` files.

Change the porosity list near the beginning:

```python
POROSITIES = [
    0.0,
    0.1,
    0.2,
    0.3,
    0.4,
    0.5,
    0.6,
]
```

The script contains a switch for the random seed:

```python
USE_FIXED_SEED = True
```

With a fixed seed, the same porous geometry is generated every time, which is
useful for reproducibility. Set

```python
USE_FIXED_SEED = False
```

to generate a new random porous geometry each time the script is run.

### `generate_oblate_shape.py`

Creates an oblate ellipsoidal `shape.dat` target. This is optional because a
homogeneous ellipsoid can also be generated directly with `ELLIPSOID` in
DDSCAT. The script is useful when an actual `shape.dat` is needed, for example
before adding porosity or multiple materials.

The semi-axes are set near the top of the script:

```python
A_X = 35
A_Y = 35
A_Z = 20
```

### `generate_two_material_sphere.py`

Creates a compact spherical `shape.dat` containing two explicitly distributed
materials. Material numbers in `shape.dat` correspond to the order of the
dielectric files in `ddscat.par`.

For two isotropic materials, `ddscat.par` has to contain for example:

```text
2 = NCOMP
'material_1.dat'
'material_2.dat'
```

and the dipoles in `shape.dat` use `1 1 1` for material 1 and `2 2 2` for
material 2.

The fraction assigned to material 2 can be changed in
`MATERIAL_2_FRACTIONS`. A fixed random seed can be used for reproducible
material distributions.

### `visualize_shape_3d.py`

Displays a custom `shape.dat` as a 3D dipole cloud. Different material numbers
are plotted separately. For very large targets, only a subset of dipoles is
shown to keep the plot responsive; this does not modify the actual target.

### `visualize_shape_slice.py`

Displays an `xy`, `xz`, or `yz` slice through `shape.dat`. This is useful for
checking the shape, porosity, or the spatial distribution of different
materials inside a grain.

### Runtime scripts

The runtime scripts read the timing information written to `ddscat.log_000`.

Available examples are:

- `runtime_vs_wavelength.py`
- `runtime_vs_porosity.py`
- `runtime_vs_oblateness.py`
- `runtime_vs_ice_fraction.py`

These scripts are intended as examples. Change the directory paths at the
beginning to match your own run structure.

---

## 7. Plotting choices

The analysis scripts contain settings such as:

```python
X_SCALE = "log"
Y_SCALE = "log"
```

Comment out one or both in case you want linear spacing (which can look clearer in many cases). 

This only changes the plot. It does not change the wavelength distribution
used by DDSCAT.

The DDSCAT wavelength grid itself is controlled separately in `input.toml`:

```toml
spacing = "LOG"
```

Available DDSCAT spacing options are:

- `LIN` — equally spaced in wavelength
- `LOG` — equally spaced in logarithmic wavelength
- `INV` — equally spaced in inverse wavelength

---

## 8. Minimal workflow

For an already compiled DDSCAT installation, the complete workflow is:

```bash
cp input.toml my_input.toml
nano my_input.toml

python3 generate_ddscat.py my_input.toml

sbatch main.sh my_input.toml

python3 check_run.py my_input.toml
python3 plot_qtable.py my_input.toml
```

For a local run without Slurm:

```bash
bash main.sh my_input.toml
```

The copied TOML input file is normally the only file that needs to be edited
for a new simulation.


---


## 9. Quickstart — Setting up a first DDSCAT run

The basic workflow is:

```text
install DDSCAT
      ↓
edit input.toml
      ↓
generate input files
      ↓
run DDSCAT
      ↓
check the run
      ↓
plot the results
```

For your first test, **do not modify the Python scripts or `ddscat.par` manually**.  
In most cases, you only need to edit a copy of `input.toml`.

---

### 1. Clone the repository

```bash
git clone <repository-url>
cd ddscat-for-beginners
```

---

### 2. Install and compile DDSCAT

DDSCAT itself is **not included** in this repository.

Download DDSCAT 7.3.4 and the example package from the official DDSCAT website.

A useful directory structure is for example:

```text
~/DDA/
├── src/
│   └── ddscat
├── examples_exp/
├── diel/
└── doc/
```

Compile DDSCAT:

```bash
cd ~/DDA/src
make ddscat
```

Check that the executable exists:

```bash
ls -l ~/DDA/src/ddscat
```

If the file is there, DDSCAT is ready to use.

---

### 3. Install the Python requirements

The helper scripts require Python 3.11 or newer.

Install the required packages:

```bash
python3 -m pip install numpy matplotlib scipy
```

---

### 4. Create your own input file

Do not edit the repository example directly. Make a copy:

```bash
cp input.toml my_input.toml
```

Then open it:

```bash
nano my_input.toml
```

or use your preferred editor.

For a first run, the most important settings are:

```toml
[paths]

ddscat_executable = "/home/USER/DDA/src/ddscat"
run_directory = "/path/to/your/ddscat_run"
material_file = "/home/USER/DDA/diel/astrosil"
```

Make sure all three paths exist.

---

### 5. Choose a simple target

For your first calculation, using DDSCAT's built-in `ELLIPSOID` target is easiest because no `shape.dat` file is required.

For example:

```toml
[target]

shape = "ELLIPSOID"
shape_parameters = [50.0, 50.0, 50.0]
```

This describes a spherical dipole target.

**Important:** `50` is **not** the grain radius in micrometres.  
It approximately describes the number of dipole spacings across the particle.

Set the actual physical grain radius separately:

```toml
[effective_radius]

minimum_um = 1.0
maximum_um = 1.0
count = 1
spacing = "LIN"
```

For example, this gives a grain with an effective radius of

```text
a_eff = 1 µm
```

---

### 6. Choose a small wavelength grid for testing

Do not start with hundreds or thousands of wavelengths.

A small first test could be:

```toml
[wavelength]

minimum_um = 1.0
maximum_um = 10.0
count = 20
spacing = "LIN"
```

Once everything works, you can increase the wavelength resolution.

For the first run, leave the numerical parameters at their defaults unless you know why you need to change them:

```toml
[numerics]

tolerance = 1.0e-5
max_iterations = 1000

solver = "PBCGS2"
fft = "GPFAFT"
polarizability = "GKDLDR"
```

---

### 7. Generate the DDSCAT input files

From the repository directory, run:

```bash
python3 generate_ddscat.py my_input.toml
```

The configured run directory should now contain at least:

```text
ddscat.par
diel.dat
```

For a `FROM_FILE` target it will additionally contain:

```text
shape.dat
```

For your first run, it is worth checking that `ddscat.par` was actually created:

```bash
cat /path/to/your/ddscat_run/ddscat.par
```

You normally do **not** need to edit this file manually.

---

### 8. Run DDSCAT

#### On a local machine (most likely for starters)

Run:

```bash
bash main.sh my_input.toml
```

#### On a Slurm cluster (useful for more expensive calculations later on)

First check the `#SBATCH` settings at the top of `main.sh` and adapt them to your cluster if necessary.

Then submit:

```bash
sbatch main.sh my_input.toml
```

Check whether the job is running:

```bash
squeue -u $USER
```

A standard DDSCAT build is serial, so requesting additional CPUs normally does not make the calculation faster!

---

### 9. Check whether the run worked

After DDSCAT finishes, run:

```bash
python3 check_run.py my_input.toml
```

For a successful run you should see something similar to:

```text
qtable rows: 20 / 20
status: DDSCAT normal termination
```

The most important part is:

```text
DDSCAT normal termination
```

If fewer wavelength rows than expected are present, the calculation probably stopped before completing all wavelengths.

A common error is:

```text
FATAL ERROR IN PROCEDURE: ZBCG2WP
ITERN>ITERMX
```

This means that the iterative solver did not converge within the allowed number of iterations.

Do not immediately increase `max_iterations`. First check the wavelength, material properties and dipole resolution.

---

### 10. Make a first plot

If the run completed successfully:

```bash
python3 plot_qtable.py my_input.toml
```

This creates:

```text
qtable_overview.png
```

inside the run directory.

The plot shows the main DDSCAT efficiency factors:

```text
Qabs   absorption
Qsca   scattering
Qext   extinction
```

with

```text
Qext = Qabs + Qsca
```

---

### What should I look at after my first successful run?

The most important DDSCAT output file for a beginner is:

```text
qtable
```

It contains wavelength-dependent quantities such as:

```text
Qext
Qabs
Qsca
g
```

More detailed information is stored in files such as:

```text
wXXXrXXX.avg
wXXXrXXXkXXX.sca
target.out
ddscat.log_000
```

You do not need to understand all of these for your first calculation.

---

### The complete beginner workflow

Once DDSCAT is installed, the workflow for most new calculations is simply:

```bash
cp input.toml my_input.toml

nano my_input.toml

python3 generate_ddscat.py my_input.toml

sbatch main.sh my_input.toml

python3 check_run.py my_input.toml

python3 plot_qtable.py my_input.toml
```

or, without Slurm:

```bash
bash main.sh my_input.toml
```

If this produces `DDSCAT normal termination` and `qtable_overview.png`, your basic DDSCAT setup works.

From here you can start experimenting with:

- different effective grain radii,
- different wavelength ranges,
- different materials,
- higher dipole resolutions,
- non-spherical targets,
- porous grains,
- or custom `shape.dat` files.

The additional scripts in `scripts/` are optional and are **not required for running DDSCAT**.

---




## 10. References and further information

- B. T. Draine & P. J. Flatau, DDSCAT User Guide
- B. T. Draine & P. J. Flatau (1994), *Discrete-dipole approximation for
  scattering calculations*, Journal of the Optical Society of America A,
  11, 1491.
- Official DDSCAT download page:
  **https://ddscat.wikidot.com/downloads**
- Optool repository:
  **https://github.com/cdominik/optool**