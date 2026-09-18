# ACO TSP Solver

A desktop application that solves the Travelling Salesman Problem with Ant Colony Optimization, visualising the search live: the best tour on a map of Bosnia and Herzegovina, the pheromone trail as edge thickness, and the convergence curve as it develops.

Built on the [natID/natGUI](https://github.com/idzafic/natID) C++ framework. University project for the Numerical Optimization course at the Faculty of Electrical Engineering (ETF), University of Sarajevo.

[![Release](https://img.shields.io/github/v/release/arminn2206/ProjNO_TSPACO_Memisevic)](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/releases)
[![Build](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/actions/workflows/release-all.yml/badge.svg)](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/actions)
[![License](https://img.shields.io/badge/license-MIT-blue)](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/blob/main/LICENSE.txt)
[![Platforms](https://img.shields.io/badge/platforms-Windows%20%7C%20macOS%20%7C%20Linux-lightgrey)](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/releases/latest)

---

## Build it in 30 seconds

The sources live in **`Implementation/`**, not at the repository root. Point your IDE or CMake at that folder:

```bash
cmake -S Implementation -B build
cmake --build build --config Release
```

Full instructions, including the Xcode and Visual Studio paths, are under [Building from source](#building-from-source).

---

## Download

Prebuilt installers for all three platforms are on the [latest release](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/releases/latest).

| Platform | File | Install |
|---|---|---|
| Windows 10/11 (x64) | `TSP_ACO-win.zip` | Unzip, run the `.exe` (or the `.msi` directly). Keep both files together. |
| macOS — Apple Silicon | `TSP_ACO-macOS-Silicon.zip` | Unzip, drag to Applications. **First launch: right-click → Open.** |
| macOS — Intel | `TSP_ACO-macOS-Intel.zip` | Same as above. |
| Linux (Ubuntu 24.04+) | `TSP_ACO-linux.zip` | Unzip, then `sudo apt install ./tspaco.deb` |

The macOS builds are ad-hoc signed rather than notarised, so Gatekeeper blocks a normal double-click on first launch. Right-click → Open once, and it will open normally from then on.

---

## What it does

The Travelling Salesman Problem asks for the shortest closed tour visiting every city exactly once — an NP-hard combinatorial optimization problem. Ant Colony Optimization attacks it with a population of artificial ants that build tours probabilistically, biased by a pheromone trail that gets reinforced in proportion to tour quality. Good edges accumulate pheromone, the colony concentrates on them, and the tours get shorter.

Each run draws 15–25 cities at random from a set of 99 real Bosnian cities with true latitude/longitude coordinates, so the distance matrix reflects actual geography rather than random points in a square.

### Features

- **Live map** — cities, the current best tour, and the pheromone trail rendered as edge thickness that thins out as the colony converges on a route
- **Convergence chart** — best-so-far cost and per-iteration best cost against iteration number, with the greedy nearest-neighbour tour drawn as a fixed reference line
- **Statistics sidebar** — iteration, best cost, runtime, greedy baseline, and the signed improvement percentage over that baseline
- **Run comparison** — the two most recently completed runs side by side with their differences, so parameter changes can actually be evaluated
- **CSV export** — two files per run: a convergence table for offline analysis, and a full problem instance (coordinates, tour, distance matrix) for verification or replay
- **Reproducible runs** — a fixed seed replays a run exactly; seed and city-draw index are both written into every export. All editable ACO parameters, the seed, and the animation speed persist across restarts (clamped the same way whether they arrive from the UI or from a previous session)
- **Bilingual UI** — English and Bosnian, chosen in Settings; the change takes effect after a restart, which the app offers to perform (95 translated strings; the in-app help below is English-only)
- **Light and dark themes** — colours adapt to the OS setting on startup
- **In-app help** — App menu → Help opens a static explanation of the algorithm, every part of the map/chart/sidebar, and every control, for anyone opening the app without this README

---

## Algorithm

The implementation follows the **Ant System** formulation from Dorigo & Stützle, *Ant Colony Optimization* (MIT Press, 2004).

**Tour construction.** From city $i$, an ant picks the next unvisited city $j$ by roulette-wheel selection over

$$p_{ij} \propto \tau_{ij}^{\alpha} \cdot \eta_{ij}^{\beta}, \qquad \eta_{ij} = 1/d_{ij}$$

where $\tau$ is pheromone intensity and $\eta$ is the distance heuristic. Ants start from randomly chosen cities each iteration — launching them all from one fixed city does not change which tours are reachable, but it correlates their early decisions, since every ant would face an identical first choice under identical pheromone.

**Global pheromone update**, applied once per iteration in three explicit steps:

$$\tau \leftarrow (1-\rho)\,\tau \qquad \Delta\tau_{ij} = \sum_k Q/L_k \qquad \tau \leftarrow \tau + \Delta\tau$$

Order matters: evaporation applies to the trail as it stood at the *start* of the iteration, and this iteration's deposits are added afterwards and are therefore not evaporated. Deposits are symmetric — $\Delta\tau_{ij}$ and $\Delta\tau_{ji}$ both receive $Q/L_k$ — since a TSP tour is undirected.

**Feasibility check.** Every constructed tour is validated as a genuine permutation of all cities before it is costed, compared against the incumbent, or allowed to deposit. An infeasible tour would be shorter than a real one and would silently become the reported best.

### Parameters

All parameters are snapshotted once when Start is pressed, so they stay constant for the duration of a run. Ants, iterations, α, β, ρ and the seed are editable in the UI; Q and τ₀ are fixed compile-time constants.

| Parameter | Symbol | Default | Effect |
|---|---|---|---|
| Number of ants | — | 20 | Tours built per iteration |
| Iterations | — | 100 | Length of the run |
| Pheromone influence | α | 1.0 | Weight on the trail |
| Heuristic influence | β | 3.0 | Weight on 1/distance |
| Evaporation rate | ρ | 0.5 | Fraction of trail lost per iteration |
| Deposit constant | Q | 100.0 | Scales the per-ant deposit (fixed, not editable) |
| Initial pheromone | τ₀ | 1.0 | Uniform starting trail (fixed, not editable) |
| Random seed | — | random | Fixed value makes the run reproducible |

### Quality baseline

Best tour cost on its own is unanchored — whether 850 km is good depends entirely on which cities were drawn. Every run is therefore measured against a **repetitive nearest-neighbour tour** over the same city set, computed once at load time by running greedy NN from every possible start and keeping the best.

The improvement percentage is displayed **signed and unclamped**. A run stopped early, or given too few ants or iterations, genuinely can fail to beat a greedy tour, and a sidebar that could only report success would be useless as an instrument.

---

## Repository layout

```
Implementation/     Application sources, resources and build files
├── CMakeLists.txt  <- point CMake / your IDE HERE, not at the repository root
├── tspaco.cmake
├── src/            C++ sources (header-only design, one class per header)
├── res/            Resources: artwork manifest, translations, map data, app icons
├── packaging/      SetupCollector configuration
└── .gitignore
Docs/               Proposal, description and presentations
```

---

## Architecture

```
Implementation/src/
├── main.cpp             Entry point
├── Application.h        natGUI application, creates the main window
├── MainWindow.h         Menu/toolbar dispatch, Start/Stop/Reset/Export/Compare
├── MainView.h           Splitter layout: map + convergence chart + stats sidebar
├── MapModel.h           ACO engine — distance & pheromone matrices, worker thread
├── ViewMap.h            Map canvas: cities, best tour, pheromone edges
├── ViewConvergence.h    Live convergence chart (primary output)
├── ViewStats.h          Run statistics sidebar
├── ViewCompare.h        Side-by-side comparison of two completed runs
├── DialogCompare.h      Modal host for the comparison view
├── ViewHelp.h           Static in-app explanation of the app (App menu → Help)
├── DialogHelp.h         Modal host for the help view
├── ViewSettings.h       Settings dialog: language, toolbar, town names, sound, trail
├── DialogSettings.h     Modal host for settings, persists to OS properties
├── RunExport.h          All file I/O — CSV export
├── Town.h               City primitive: coordinates, name, draw state
├── Primitive.h          Base drawable with visit state
├── GraphType.h          City index type
├── Constants.h          Defaults, thresholds, tuning constants
├── MenuBar.h            Menu definitions
└── ToolBar.h            Toolbar, mirrors every menu action
```

### Matrices

Both the $N \times N$ distance matrix and the $N \times N$ pheromone matrix are natID `dense::DblMatrix` instances. The whole-matrix steps of the update rule use the library's own operators — evaporation is `operator*=`, the deposit application is `operator+=` — rather than hand-written nested loops, which lets the matrix class decide how to traverse its own storage.

The pheromone matrix's diagonal is deliberately left at zero rather than seeded with τ₀. A self-loop is never a legal move, so the diagonal is never read during tour construction; seeded with τ₀ it would instead enter evaporation and drift, silently contributing to the L1 norm reported in the export.

### Threading

The ACO loop runs on a `std::thread` so the UI stays responsive. The model owns a mutex guarding the published state — best tour, cost history, pheromone snapshot — and the worker publishes consistent snapshots that the UI polls once per frame. Widget updates are marshalled back to the main thread with `asyncExecInMainThread`. The worker never touches a widget directly.

The pheromone matrix is worker-owned and unlocked during the update itself, which is safe precisely because no other thread reads it: the UI reads a separate mutex-guarded snapshot published after each complete update, so it can never observe the trail mid-evaporation.

---

## Building from source

### Prerequisites

- [natID SDK](https://github.com/idzafic/natID) cloned to `~/natID.SDK` (and `~/natID.Utils`), with the prebuilt binaries for your platform extracted into `natID.SDK/bin` per that folder's `ReadMe.txt`
- CMake 3.18 or newer (tested through CMake 4.4)
- A C++20 compiler — MSVC 2022+, AppleClang (Xcode 14+), or GCC 13+
- Linux only: `libgtk-4-dev libadwaita-1-dev libopenal-dev`

> **The source folder is `Implementation/`, not the repository root.** Every path below reflects that. If an IDE reports that it cannot find a `CMakeLists.txt`, it was pointed at the repository root.

### macOS — Xcode

```bash
git clone https://github.com/arminn2206/ProjNO_TSPACO_Memisevic.git
cd ProjNO_TSPACO_Memisevic
cmake -S Implementation -B ~/build-tspaco -G Xcode
open ~/build-tspaco/tspaco.xcodeproj
```

The same thing through the CMake GUI, following the natID book's macOS setup:

1. *Where is the source code* → the **`Implementation`** folder inside the clone
2. *Where to build the binaries* → any separate folder, e.g. `/Users/<you>/build-tspaco`
3. **Configure** → choose the **Xcode** generator → **Generate** → **Open Project**
4. In Xcode: Product → Scheme → Edit Scheme → Run → **Build Configuration: Release**
5. ⌘B to build, ⌘R to run

Xcode is a multi-configuration generator, so `-DCMAKE_BUILD_TYPE` has no effect on it — Debug vs Release is chosen inside Xcode, in the scheme.

### macOS / Linux — command line

```bash
cmake -S Implementation -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

### Windows — Visual Studio

```
cmake -S Implementation -B build
cmake --build build --config Release
```

If you prefer the IDE, use **File → Open → Folder** and select the **`Implementation`** folder. Visual Studio looks for `CMakeLists.txt` in exactly the folder you open and will do nothing if given the repository root. Add an `x64-Release` configuration through *Manage Configurations*; the repository intentionally ships no `CMakeSettings.json`, because the IDE's own CMake integration has been observed to produce a Debug binary while the configuration dropdown reads Release. When in doubt, build Release from the command line as shown above and check the build log rather than the dropdown.

### Where the binary lands

The SDK redirects build output away from the source tree:

```
~/natID.RAMDisk/Out/tspaco/Release/tspaco.app     (macOS)
~/natID.RAMDisk/Out/tspaco/Release/tspaco.exe     (Windows)
~/natID.RAMDisk/Out/tspaco/Release/tspaco         (Linux)
```

An empty build folder next to the sources does not mean the build failed.

### Packaging

Installers are produced by the SDK's `SetupCollector` tool against [`Implementation/packaging/tspaco.xml`](packaging/tspaco.xml):

```bash
SetupCollector <path-to-setups>/tspaco.xml
```

`Implementation/packaging/GTK4.xml` overrides the SDK's own GTK package definition — the stock file points at `$MyBin/GTK/release/bin`, but the Windows SDK ships its GTK runtime DLLs flat in `$MyBin/GTK`.

---

## Continuous integration

[`.github/workflows/release-all.yml`](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/blob/main/.github/workflows/release-all.yml) (on the `main` branch) builds installers for all four targets — Windows, macOS ARM, macOS Intel, Linux — on every `v*` tag, and publishes them to a GitHub Release.

Each job installs the natID SDK from scratch, downloads the matching prebuilt binaries, builds in Release, runs `SetupCollector`, and uploads the result. Two environment quirks are handled: the SDK expects a RAM disk at a platform-specific mount point (`R:` / `/Volumes/RAMDisk` / `/media/RAMDisk`), faked with `subst` or a symlink, and the packaging config expects the sources at a fixed path, provided by symlinking the checkout.

The workflow can also be run manually from the Actions tab with `publish_release: no` to verify a build without creating a release.

---

## Project context

Implemented for the Numerical Optimization course at ETF Sarajevo, extending the professor's `B_S03_Maps` natID example. The map rendering, canvas animation loop, and background-thread architecture come from that example; cities replace towns, TSP tour edges replace roads, and the threaded loop drives the ACO iteration engine.

The original `Graph` class from the example was dropped: ACO needs a complete graph as an $N \times N$ matrix for pheromone and distance lookups, not the adjacency-list structure a shortest-path search wants.

**Author:** Armin Memišević (index 20016)
**Theoretical reference:** Marco Dorigo & Thomas Stützle, *Ant Colony Optimization*, MIT Press, 2004

---

## License

MIT — see [LICENSE.txt](https://github.com/arminn2206/ProjNO_TSPACO_Memisevic/blob/main/LICENSE.txt).

Built on the natID/natGUI framework and adapted from the `B_S03_Maps` example, both by [prof. Izudin Džafić](https://github.com/idzafic), used with permission.
