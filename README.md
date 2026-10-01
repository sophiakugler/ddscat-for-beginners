# DDSCAT for Beginners

This repository contains a small collection of scripts for setting up, running, and analysing DDSCAT calculations. It is intended as a practical starting point for users who have never used DDSCAT before. You do not need to understand the DDSCAT Fortran source code to use this repository.

For a first calculation, you normally only need to edit one file:

```text
input.toml
```

The basic workflow is:

```text
install DDSCAT
      ↓
edit input.toml
      ↓
run main.sh
      ↓
check the calculation
      ↓
plot the result
```

---

## 1. What is DDSCAT?

DDSCAT (**Discrete Dipole SCATtering**) calculates how light interacts with particles using the Discrete Dipole Approximation (DDA).

Very roughly, DDSCAT replaces a particle by many small dipoles:

```text
real particle
      ↓
many small dipoles
      ↓
interaction with light
      ↓
absorption + scattering
```

More dipoles usually give a better representation of the particle, but also require more computation time. For basic DDSCAT calculations, in-depth understanding of the mathematical details is not needed.

A more detailed introduction is available here:

**[DDSCAT for beginners (PDF)](ddscat_for_beginners.pdf)**

---

# 2. Installation

## 2.1 Clone this repository

```bash
git clone <repository-url>
cd ddscat-for-beginners
```

---

## 2.2 Install DDSCAT

DDSCAT itself is not included in this repository. Download DDSCAT 7.3.4 and the example package from:

**https://ddscat.wikidot.com/downloads**

Download:

1. **DDSCAT 7.3.4 FORTRAN code**
2. **DDSCAT 7.3.4 Examples**

A useful directory structure is:

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

If the file exists, DDSCAT is ready.

---

## 2.3 Install Python requirements

The helper scripts require Python 3.11 or newer.

Install:

```bash
python3 -m pip install numpy matplotlib scipy
```

---

# 3. A first DDSCAT calculation

For your first run, make a copy of the input.toml file:

```bash
cp input.toml my_input.toml
```

Edit it 

```bash
nano my_input.toml
```

or use any other text editor.

---

# 4. Edit `input.toml`

There are four things you need to understand for your first run:

1. Where DDSCAT is installed
2. Where the results should be written
3. Which material should be used
4. Which particle and wavelengths should be calculated

---

## 4.1 Paths

Find:

```toml
[paths]

ddscat_executable = "/home/USER/DDA/src/ddscat"
run_directory = "/path/to/your/ddscat_run"
material_file = "/home/USER/DDA/diel/astrosil"
```

Change these paths for your system.

### `ddscat_executable`

This is the compiled DDSCAT program:

```text
~/DDA/src/ddscat
```

### `run_directory`

This is where DDSCAT will write the calculation.

For example:

```text
/home/USER/ddscat_runs/test
```

### `material_file`

This contains the optical properties of the material.

For example:

```text
~/DDA/diel/astrosil
```

---

# 5. Choose a particle

For the first test, use DDSCAT's built-in `ELLIPSOID` target. You do not need a separate `shape.dat` file for this.

Use:

```toml
[target]

shape = "ELLIPSOID"
shape_parameters = [50.0, 50.0, 50.0]
```

Because all three numbers are identical, this produces a spherical target.

## Important

```text
50
```

does **not** mean

```text
50 µm
```

It roughly describes how many dipole spacings fit across the particle. The physical size of the particle is set separately.

---

# 6. Choose the particle size

For example:

```toml
[effective_radius]

minimum_um = 1.0
maximum_um = 1.0
count = 1
spacing = "LIN"
```

This calculates one particle with

```text
a_eff = 1 µm
```

For a compact sphere, you can think of `a_eff` approximately as the particle radius.

---

# 7. Choose the wavelengths

For your first test, use only a small number of wavelengths.

For example:

```toml
[wavelength]

minimum_um = 1.0
maximum_um = 10.0
count = 20
spacing = "LIN"
```

This calculates 20 wavelengths between 1 and 10 µm. Do not start with hundreds or thousands of wavelengths, first make sure that the basic calculation works!

---

# 8. Leave the numerical settings alone

For your first run, keep:

```toml
[numerics]

tolerance = 1.0e-5
max_iterations = 1000

solver = "PBCGS2"
fft = "GPFAFT"
polarizability = "GKDLDR"
```

You normally do not need to change these settings for a first test.

---

# 9. Run DDSCAT

You now have two options.

## Local machine

Run:

```bash
bash main.sh my_input.toml
```

## Slurm cluster

Run:

```bash
sbatch main.sh my_input.toml
```

You can check your Slurm job with:

```bash
squeue -u $USER
```

The `#SBATCH` settings are located at the top of `main.sh`. A normal serial DDSCAT executable uses one CPU; requesting more CPUs does not automatically make DDSCAT faster!

---

# 10. What does `main.sh` actually do?

You do not normally need to run the individual steps yourself.

`main.sh` takes care of the workflow:

```text
my_input.toml
      ↓
generate_ddscat.py
      ↓
ddscat.par
      ↓
DDSCAT
      ↓
qtable + log files + other outputs
```

In other words:

### You edit

```text
my_input.toml
```

### The repository generates

```text
ddscat.par
```

### DDSCAT generates

```text
qtable
ddscat.log_000
target.out
*.avg
*.sca
...
```

---

# 11. Check whether the run worked

After DDSCAT has finished, run:

```bash
python3 check_run.py my_input.toml
```

For the example with 20 wavelengths, a successful calculation should look approximately like:

```text
qtable rows: 20 / 20
status: DDSCAT normal termination
```

The most important message is:

```text
DDSCAT normal termination
```

That means DDSCAT finished successfully.

---

# 12. A common error

You may see:

```text
FATAL ERROR IN PROCEDURE: ZBCG2WP
ITERN>ITERMX
```

This means the iterative solver did not converge before reaching the maximum number of iterations. Do not straight away increase `max_iterations`, first check:

- wavelength
- material
- particle size
- dipole resolution

---

# 13. Plot the result

After a successful run:

```bash
python3 plot_qtable.py my_input.toml
```

This creates:

```text
qtable_overview.png
```

inside the run directory.

The plot shows:

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

# 14. The most important DDSCAT output

For a beginner, the most important output file is:

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

Other DDSCAT output files include:

```text
wXXXrXXX.avg
wXXXrXXXkXXX.sca
target.out
ddscat.log_000
```

You do not need to understand all of them for your first calculation.

---

# 15. Complete beginner workflow

Once DDSCAT is installed, a normal calculation is essentially:

```bash
cp input.toml my_input.toml

nano my_input.toml

bash main.sh my_input.toml

python3 check_run.py my_input.toml

python3 plot_qtable.py my_input.toml
```

or on a Slurm cluster:

```bash
cp input.toml my_input.toml

nano my_input.toml

sbatch main.sh my_input.toml

python3 check_run.py my_input.toml

python3 plot_qtable.py my_input.toml
```

If you get

```text
DDSCAT normal termination
```

and

```text
qtable_overview.png
```

your basic setup works.

---

# 16. What can I change next?

Once the simple test works, you can start changing:

- particle radius
- wavelength range
- material
- dipole resolution
- particle shape
- porosity
- material composition
- custom `shape.dat` targets

Change one thing at a time when you are learning DDSCAT. In this way you can easy keep track of what changing one parameter actually does to the output.

---

# 17. Custom `shape.dat` files

You only need this section once the simple `ELLIPSOID` example works.

Instead of:

```toml
shape = "ELLIPSOID"
```

you can use:

```toml
shape = "FROM_FILE"
shape_file = "/path/to/shape.dat"
```

The helper script copies the geometry into the run directory as:

```text
shape.dat
```

For a custom target, the run directory therefore contains approximately:

```text
ddscat.par
diel.dat
shape.dat
```

For `ELLIPSOID`, no `shape.dat` is required.

---

# 18. Multiple materials

For targets containing more than one material, DDSCAT needs one dielectric file for each material.

For example:

```text
2 = NCOMP

'material_1.dat'
'material_2.dat'
```

The material numbers inside `shape.dat` correspond to this order.

For two isotropic materials:

```text
1 1 1
```

means material 1 and

```text
2 2 2
```

means material 2.

See:

```text
scripts/generate_two_material_sphere.py
```

for an example.

---

# 19. Optional scripts

Everything inside:

```text
scripts/
```

is optional. You do not need these scripts to perform a basic DDSCAT calculation. They provide examples for more specialised tasks!

## `compare_qtables.py`

Compares two DDSCAT calculations.

Useful for comparing things such as:

- particle shapes
- porosities
- compositions
- dipole resolutions

---

## `plot_multiple_qtables.py`

Plots several DDSCAT calculations in one figure.

---

## `generate_porous_sphere.py`

Generates porous spherical `shape.dat` targets.

---

## `generate_oblate_shape.py`

Generates an oblate ellipsoidal `shape.dat`. For a homogeneous ellipsoid, using DDSCAT's built-in `ELLIPSOID` target is usually easier.

---

## `generate_two_material_sphere.py`

Generates a spherical target containing two materials.

---

## `visualize_shape_3d.py`

Displays a `shape.dat` target as a 3D dipole cloud.

---

## `visualize_shape_slice.py`

Displays a slice through a `shape.dat` target.

Useful for inspecting:

- particle shape
- porosity
- material distribution

---

## Runtime scripts

Runtime analysis examples include:

```text
runtime_vs_wavelength.py
runtime_vs_porosity.py
runtime_vs_oblateness.py
runtime_vs_ice_fraction.py
```

These read timing information from:

```text
ddscat.log_000
```

They are examples and may need to be adapted to your directory structure.

---

# 20. Repository structure

The most important files are:

```text
.
├── README.md
├── input.toml
├── generate_ddscat.py
├── main.sh
├── check_run.py
├── plot_qtable.py
├── ddscat_for_beginners.pdf
└── scripts/
```

For normal use:

```text
input.toml
```

is the file you edit.

```text
generate_ddscat.py
```

creates the DDSCAT input.

```text
main.sh
```

runs the calculation.

```text
check_run.py
```

checks whether it worked.

```text
plot_qtable.py
```

plots the result. Everything else can wait until you need it.

---

# 21. References

- B. T. Draine & P. J. Flatau, DDSCAT User Guide
- B. T. Draine & P. J. Flatau (1994), *Discrete-dipole approximation for scattering calculations*, Journal of the Optical Society of America A, 11, 1491.
- Official DDSCAT download page:  
  **https://ddscat.wikidot.com/downloads**
- Optool repository:  
  **https://github.com/cdominik/optool**

---

> **Note:** Some of the Python and shell code in this repository was developed with assistance from large language models, including ChatGPT (OpenAI). The scripts were reviewed, modified, and tested on the calculations used during this project. Paths and numerical parameters should still be checked before using them for a new setup.