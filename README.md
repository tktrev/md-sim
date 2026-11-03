# NBMD1 — Molecular Dynamics Simulator for an Argon Crystal

NBMD1 is a legacy Windows Forms application for classical molecular dynamics (MD) simulation of an argon crystal. It initializes a cubic or face-centered cubic (FCC) lattice, evolves the particles with a Lennard-Jones potential, applies stepwise heating and a thermostat, and reports structural and thermodynamic quantities in real time.

The program was originally written as a research/student project and is published here mainly for archival purposes. It targets .NET Framework 4.0 and uses a Russian-language interface.

![NBMD1 main window](images/image.png)

## Overview

The application provides an interactive environment for watching an argon particle system evolve. Starting from an ideal lattice, it heats the crystal through a sequence of target temperatures and tracks:

- Total, kinetic, and potential energy.
- Instantaneous and time-averaged temperature.
- The radial distribution function (RDF).
- Lattice period estimated from the RDF peak and from the particle number density.
- First-shell coordination number.
- The temperature and particle count of a dense "drop" and the surrounding gas.
- A two-dimensional projection or selected slice of the particle configuration.

The main simulation loop runs inside the UI thread, so the window remains responsive through `Application.DoEvents()`. Results are displayed while the calculation is running and can also be saved to text files.

## Key Implementation Highlights

- **Complete MD engine:** Implements cubic and FCC lattice initialization, center-of-mass centering, pairwise Lennard-Jones forces, potential-energy accumulation, and reflecting wall boundaries in `MyFuncProc.cs`.
- **Efficient neighbor search:** Uses a cell-linked list with forward neighbor-cell offsets to avoid checking every particle pair. Force calculation and RDF histogram accumulation share the same neighbor traversal.
- **Time evolution and temperature control:** Runs a semi-implicit Euler integration loop with momentum correction, stepwise heating, and both hard and Berendsen-style velocity rescaling in `Form1.cs`.
- **Structural analysis:** Calculates the radial distribution function, fits a cubic spline around its first peak, extracts the first-shell coordination number, and estimates the lattice period from both the RDF peak and local particle density using `MySpline.cs` and `MyFuncProc.cs`.
- **Drop/gas diagnostics:** Detects the densest cell to define a drop center, then reports separate temperatures and particle counts for the drop and surrounding gas in `Find_Drop`.
- **Interactive visualization:** Provides a live `x-y` particle projection, optional layer slicing, RDF plotting through ZedGraph, and dedicated panels for energy, temperature, and experiment settings.
- **Persistence and averaging:** Saves and loads particle configurations, periodically writes configuration/RDF/parameter files, and maintains running averages over simulation steps through `Dataworks.cs`.
- **Numerical support infrastructure:** Includes reusable 3D vector types in `D3.cs`, cubic-spline interpolation in `MySpline.cs`, and bundled ALGLIB routines for random-number generation and other numerical utilities.

## Physical Model

### Particles and units

The default system parameters are defined in `Vars.cs`:

| Parameter | Default | Meaning in the code |
| --- | ---: | --- |
| Particle mass | `40` | Mass used in the equations of motion |
| Initial lattice period | `4` | Distance between lattice sites |
| Lennard-Jones minimum distance | `3.8` | `rMin` in the potential |
| Lennard-Jones energy scale | `0.0123` | `Epsil`, in eV |
| Boltzmann constant | `8.617342791e-5` | eV/K |
| MD time step | `0.5` | Listed as `dt` in the main loop |
| Initial target temperature | `60` | Kelvin |

The code assumes length values around angstroms and uses the Boltzmann constant in eV/K for its temperature conversion. There is no separate unit-conversion layer; the velocity and time-step values are applied directly by the integrator, so the saved velocities and the reported simulation time should not be interpreted as a fully calibrated physical unit system.

### Initial configuration

`MyFunсProc.CreateLattice` supports two lattice types:

- `type == 0`: a simple cubic lattice.
- `type == 1`: an FCC lattice. The **Start** command currently uses this type.

With the default `6 x 6 x 6` grid in `Vars.cs`, the FCC construction creates 666 particles (216 corner particles plus 450 face-centered positions). The initial velocities are zero, and the coordinate system is translated so that its origin is at the center of mass.

The simulation box is a finite cube of size `Cube_Size = 20 * part_amount_x * Lattice_period` (480 code units by default). It is not periodic. Particles crossing a wall are reflected back into the box, and their wall-normal velocity is rescaled toward the current target temperature.

### Interaction potential

The force calculation uses the Lennard-Jones expression implemented in `Find_Force`:

```text
U(r) = Epsil * ((rMin / r)^12 - 2 * (rMin / r)^6)
```

Here `rMin` is the separation at the potential minimum. The force on each particle is accumulated pairwise, and the total potential energy `U_system` is accumulated at the same time. The code uses a cell-linked-list scheme (`HOC`, `LL`, and the 13 forward neighbor-cell offsets plus the current cell in `NC`) to avoid checking every particle pair. Pair separation is also binned into a per-particle RDF histogram during the same neighbor traversal.

The interaction has no energy shift or explicit smooth cutoff in the force loop; the finite neighbor-cell search effectively limits which pairs are visited. This is important when interpreting quantitative results.

### Time integration and momentum correction

Each MD step performs an explicit update in the following order:

1. Build the cell lists for the current coordinates.
2. Clear the RDF accumulators.
3. Compute forces and potential energy.
4. Update velocities: `v += (F / mass) * dt`.
5. Remove the center-of-mass momentum.
6. Apply the thermostat.
7. Update positions: `x += v * dt`.
8. Apply reflecting wall conditions.

Because velocity is updated before position using the newly computed force, this is a semi-implicit (symplectic) Euler integrator rather than velocity Verlet. The fixed step is applied directly and has to be chosen small enough for the potential and temperature being simulated.

### Heating and thermostat

The application supports two connected temperature-control mechanisms:

- A Berendsen-style velocity rescaling with relaxation time `Time_relax`.
- An immediate "hard" velocity rescaling to the target temperature.

The initial state uses hard rescaling. Every `Step_Temp` steps the loop switches to the Berendsen correction, and at each multiple of `HS_Corr_ratio * Step_Temp` it switches back to hard correction for the next `Step_Temp` steps. When heating is enabled, the target temperature is increased by `Temp_plus` at those same `HS_Corr_ratio * Step_Temp` boundaries. With the defaults in `Vars.cs` (`HS_Corr_ratio = 5000`, `Step_Temp = 10000`), a temperature plateau is 50,000,000 MD steps, which is very large for an interactive session; these values should be reduced for practical runs unless that schedule is intentional.

### Drop/gas analysis

`Find_Drop` divides the configuration into a dense drop and a surrounding gas:

1. A cell grid is rebuilt and the cell with the largest particle occupancy is selected.
2. Its position defines the center of the drop.
3. Particles within `Drop_size = part_amount_x * Lattice_period` (24 by default) of that center are counted as drop particles; the rest are counted as gas particles.
4. Separate kinetic temperatures and particle counts are reported for both populations.

This is a geometric diagnostic, not a full phase-identification or clustering algorithm.

### RDF and lattice estimates

The radial distribution function is accumulated in bins of width `delta_R = 0.1` over the range `0..R_range` (8 by default), giving 81 radial samples. It is an unnormalized neighbor-count histogram `N(R)` rather than a density-normalized `g(R)`, and distances beyond the configured range are clamped into the final bin. Only particles inside the central sphere of radius `R_cut = part_amount_x * Lattice_period / 3` (8 by default) are included when the histogram is averaged. This deliberately reduces surface effects from the finite box.

`MySpline` builds a cubic spline around the first RDF peak, finds the interpolated maximum, and locates the following minimum. The maximum position is used to estimate the nearest-neighbor distance, and the integrated RDF up to the minimum gives the first-shell coordination number.

A second lattice-period estimate is obtained from the local number density. `Find_Lattice_Size` picks one radius between `R_c_Min` and `R_c_Max`, counts particles inside that sphere, and infers a lattice period from the assumed four atoms per FCC unit cell. It repeats that same estimate 100 times and averages the results; despite the loop count, it does not sample 100 different radii because the random value is generated outside the loop.

## Simulation Workflow

The main loop is implemented in the **Start** handler in `Form1.cs`:

1. Create the FCC lattice and center it at the origin.
2. Build the cell-list neighbor structure.
3. Reset the MD step counter and start the simulation clock.
4. On every step:
   - Rebuild the neighbor cells.
   - Compute forces, potential energy, and RDF pair counts.
   - Integrate velocities and positions.
   - Correct the total momentum.
   - Apply the heating schedule and thermostat.
   - Reflect particles at the cube walls.
   - Compute drop/gas temperatures and particle counts.
   - Update the running averages for temperature, energy, RDF, lattice period, and coordination number.
5. Every `Step_Freq` steps:
   - Refresh the energy, temperature, RDF, and configuration displays.
   - Store one averaged sample in `Param1`.
6. Every `Param_Save_Freq * Step_Freq` steps (10,000 by default):
   - Write the particle configuration, RDF, and parameter table to disk.
   - Reset the sample buffer.

The **Stop** command sets `Vars.Start = false`, which ends the loop after the current step. There is no separate pause/resume state.

## Using the Application

The interface is in Russian. The main controls are:

| Menu or control | Function |
| --- | --- |
| **Файл → Загрузить** | Load a particle configuration from a text file |
| **Файл → Сохранить** | Save the current step, particle count, velocities, and positions |
| **Файл → Выход** | Close the application |
| **МД-цикл → Старт** | Initialize a new FCC system and run the MD loop |
| **МД-цикл → Стоп** | Stop the loop |
| **Система → Температура** | Set the target temperature |
| **Система → Время релаксации** | Set the thermostat relaxation time |
| **Система → Нагрев → Шаг температуры** | Set the temperature increment |
| **Система → Нагрев → Шаг по времени** | Set the number of MD steps between temperature changes |
| **Система → Нагрев → Запустить нагрев** | Enable or disable stepwise heating |
| **Рисование → Рисовать** | Show or hide particles |
| **Рисование → Рисовать по слоям** | Show only the layer selected by the slider |
| **Рисование → Рисовать область расчёта РФР** | Draw the RDF sampling sphere |
| **Рисование → Отображение области капли** | Draw the drop circle |
| **График** | Enable or disable the RDF plot |
| **SaveRDF** | Save the current RDF table |

The particle panel shows an `x-y` projection of the three-dimensional configuration. The slider selects a layer along `z` when layer drawing is enabled. The right-hand lists show energy, RDF-derived quantities, drop/gas temperatures, and the current experiment settings. The ZedGraph panel shows the averaged RDF.

### Important behavior of Load

**Загрузить** reads a configuration into the existing particle arrays and sets the internal `Start` flag, but it does not itself enter the MD loop or restore thermostat/simulation settings. Pressing **Старт** afterward calls `CreateLattice` and starts a fresh FCC system, so the load command should be treated as a configuration importer rather than a full checkpoint-resume feature.

## Configuration File Format

Saved configurations are plain text files with two header lines followed by one tab-separated line per particle:

```text
<MD step number>
<particle count>
<index> <vx> <vy> <vz> <x> <y> <z>
...
```

The save command writes velocities before coordinates. The loader expects the same layout and uses the second line to determine the particle count. The file does not contain the temperature, heating schedule, time step, lattice parameters, or other simulation settings.

## Output Files

Automatic output is written by `Dataworks.cs` to hard-coded paths on Windows:

```text
C:\4K4\CsProg\NBMD1\Autosaves\Config\Conf_<step>.txt
C:\4K4\CsProg\NBMD1\Autosaves\Parameters\Param_<timestamp>.txt
C:\4K4\CsProg\NBMD1\Autosaves\RDF\RDF_<step>.txt
```

- `Conf_<step>.txt` uses the configuration format described above.
- `RDF_<step>.txt` contains the step number, particle count, and rows of `bin_index`, `radius`, and `N(radius)`.
- `Param_<timestamp>.txt` is a tab-separated table with a Russian header and columns for step, temperature, energy, lattice period from density, lattice period from the RDF peak, coordination number, drop/gas temperatures, and drop/gas particle counts.

The `Autosaves` directories are not created by the current code (the directory-creation call is commented out), and the paths are not configurable in the UI. On a different machine these paths must be changed in `Dataworks.cs`, and the target directories must already exist and be writable.

## Project Structure

| File | Responsibility |
| --- | --- |
| `Program.cs` | WinForms application entry point |
| `Form1.cs` | UI event handlers, simulation loop, plotting, save/load |
| `Form1.Designer.cs` | Generated WinForms layout and control declarations |
| `Vars.cs` | Global simulation state, physical constants, arrays, and tunable parameters |
| `MyFuncProc.cs` | Lattice creation, cell lists, forces, momentum correction, RDF, density estimate, drop/gas analysis, wall reflection |
| `MySpline.cs` | Cubic-spline construction, interpolation, and RDF peak search |
| `D3.cs` | `Double3` and `Int3` vector types plus the `MyParameters` sample structure |
| `MNK2.cs` | Least-squares parabola helper (present but not used by the active loop) |
| `Dataworks.cs` | Automatic text-file output and averaging-reset helpers |
| `Archives.cs` | Archived experiments and commented-out alternative force/error code |
| `Alglib/` | Bundled ALGLIB numerical library; the active loop uses its random-number generator |

## Building and Running

Requirements:

- Windows.
- Visual Studio with .NET Framework 4.0 support, or an equivalent MSBuild toolchain.
- The ZedGraph 5.1.5 WinForms assembly.

Open `basic-md-sim.sln` in Visual Studio and build the `NBMD1` project for `x86`. The project targets .NET Framework 4.0 and references `ZedGraph.dll` and `ZedGraph.Web.dll` through this relative path:

```text
..\..\..\zedgraph_dll_v515\zedgraph_dll_v5.1.5\
```

Those DLLs are not included in this repository. Either place them at that location, or update the `HintPath` entries in `NBMD1.csproj` to point to a local ZedGraph installation. The project is configured for `x86`, with build output written to `bin\Debug` or `bin\Release`.

The application is a Windows-only WinForms `.exe`; it is not a cross-platform .NET Core/.NET application.

## Archival Notes and Limitations

This repository preserves the original implementation rather than presenting a validated modern simulation package. In particular:

- The UI and most comments are in Russian.
- The project targets the obsolete .NET Framework 4.0 and has an external, non-bundled ZedGraph dependency.
- Automatic save paths are absolute Windows paths, and their directories are not created automatically.
- The force loop does not apply an explicit cutoff or potential shift.
- The density-based lattice estimate reuses one random probe radius for its entire averaging loop.
- Integration uses semi-implicit Euler, not a higher-order Verlet scheme.
- The default heating plateau is extremely long for interactive use.
- Loading a configuration does not restore the run settings and is not a true resume mechanism.
- Physical results depend on the hard-coded parameter set in `Vars.cs`; the program does not provide a units or parameter-validation layer.

For new work, these points should be addressed before treating the code as a production research tool or using it for quantitative predictions.

## License

This project is distributed under the GNU General Public License, version 3 or later. See `LICENSE` for the full license text.

### Why this license

The repository bundles ALGLIB 3.10.0 under the GNU General Public License, version 2 or later. Distributing a combined work that includes GPL-covered code requires the combined distribution to comply with the GPL. GPL-3.0 is compatible with ALGLIB's "version 2 or later" terms and adds clearer patent, termination, and redistribution requirements.

If you distribute binaries of this application, the GPL requires that recipients also be able to obtain the corresponding source code and that they receive the applicable license notices. The external ZedGraph dependency is not bundled here and has its own licensing terms; review those terms before redistribution.

If the project owner is the sole copyright holder of all non-ALGLIB code, another option is to replace or remove the bundled ALGLIB sources and then choose a permissive license such as MIT. That is a larger change and would require removing the dependency on `alglib.hqrndstate` and its random-number routines.
