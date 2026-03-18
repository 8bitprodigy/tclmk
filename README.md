# tclmk

A lightweight, project-file-driven build system with an optional Tk GUI.
No dependencies beyond a standard Tcl/Tk installation — which ships with
virtually every Linux distribution, the BSDs, and macOS out of the box.

---

## Tools

| File       | Purpose                                                       |
| ---------- | ------------------------------------------------------------- |
| `tclmk`    | Build tool — headless or GUI                                  |
| `tclmkcfg` | Project configurator GUI — creates and edits `.tclproj` files |

Both files are self-contained executable scripts. Drop them anywhere on your
`PATH` (e.g. `/usr/local/bin`) or keep them in your project directory.

---

## Requirements

- **Tcl 8.6** or later (`tclsh`)
- **Tk 8.6** or later (`wish`) — optional, required for the GUI

Both are available in all major package managers:

```sh
# Debian / Ubuntu
sudo apt install tcl tk

# Fedora / RHEL
sudo dnf install tcl tk

# Arch
sudo pacman -S tcl tk

# FreeBSD
sudo pkg install tcl86 tk86
```

---

## Quick Start

### 1. Create a project

Run `tclmkcfg` to open the configurator and create a `.tclproj` file, or
write one by hand — it is plain Tcl and the format is documented in
`tclproj-spec.md`.

### 2. Build

```sh
# Open the GUI (requires Tk)
tclmk

# Build headless for a specific platform
tclmk --nogui linux-gcc -O2 -DRELEASE

# Build multiple platforms at once
tclmk --nogui linux-gcc -O3 dreamcast-kos -O2

# Build a specific target within a platform
tclmk --nogui dreamcast-kos:cdi -O2
```

### 3. Install system-wide (optional)

```sh
sudo install -m 755 tclmk tclmkcfg /usr/local/bin/
```

Once installed, run `tclmk` from any project directory that contains a
`build.tclproj` file.

---

## tclmk

### GUI Mode

Running `tclmk` with no arguments opens the GUI if Tk is available.

```
┌─────────────────────────────────────────────────┐
│  tclmk — MyProject 0.1.0                        │
├─────────────────────────────────────────────────┤
│ [Platforms] [linux-gcc] [dreamcast-kos]         │
├─────────────────────────────────────────────────┤
│  Build for:                                     │
│    ☑ linux-gcc                                  │
│    ☑ dreamcast-kos                              │
│                                                 │
│  ┌─ Build Log ───────────────────────────────┐  │
│  │ [CC] src/main.c                           │  │
│  │ [LD] build/linux-gcc/mygame               │  │
│  └───────────────────────────────────────────┘  │
├─────────────────────────────────────────────────┤
│  [████████████░░░░░] Compiling src/core/foo.c   │
├─────────────────────────────────────────────────┤
│[Cancel]        [Clean] [Export Makefile] [Build]│
└─────────────────────────────────────────────────┘
```

Check a platform's checkbox to enable it for building. Checking a platform
adds a tab for that platform where you can set per-platform build options:
optimisation level, debug flags, extra defines, extra CFLAGS, and output
directory. If a platform defines multiple targets (e.g. `binary` and `cdi`),
target checkboxes appear at the top of its tab.

If the project has git submodule dependencies, a **Build** radio button
appears above the progress bar. Selecting **Dependencies** switches the view
to a dependency list where you can select individual deps to rebuild and pull
the latest version of each.

### Headless Mode

```sh
tclmk --nogui [options] [platform[:target] [flags...] ...]
```

**Options:**

| Flag             | Description                                            |
| ---------------- | ------------------------------------------------------ |
| `--nogui`        | Force headless mode                                    |
| `--clean`        | Clean build artifacts                                  |
| `--export`       | Export a Makefile                                      |
| `--list`         | List platforms, targets, and flags                     |
| `--deps`         | Rebuild all dependencies                               |
| `--dep=NAME`     | Rebuild a specific dependency                          |
| `--pull=NAME`    | Pull latest for a dependency then rebuild it           |
| `--pull-all`     | Pull latest for all dependencies then rebuild          |
| `--project FILE` | Use a specific project file (default: `build.tclproj`) |
| `--help`         | Show usage                                             |

**Examples:**

```sh
# List all platforms and their available flags
tclmk --list

# Build linux-gcc with release flags
tclmk --nogui linux-gcc -O3 -DRELEASE

# Build two platforms in one invocation
tclmk --nogui linux-gcc -O3 dreamcast-kos -O2

# Build only the CDI packaging target for Dreamcast
tclmk --nogui dreamcast-kos:cdi -O2

# Export a Makefile for distro packaging
tclmk --export linux-gcc

# Rebuild a specific submodule dependency
tclmk --nogui --dep=SDL2

# Pull and rebuild all submodule dependencies
tclmk --nogui --pull-all
```

Running `tclmk --nogui` with no platform arguments prints usage and exits.
Running `tclmk` with no arguments and no Tk available also prints usage.

### Project File

`tclmk` looks for `build.tclproj` in the current working directory. Specify
an alternate file with `--project`:

```sh
tclmk --nogui --project /path/to/other.tclproj linux-gcc
```

### Build Output

By default, build output is written to `build/<toolchain-name>/`:

```
build/
  linux-gcc/
    obj/          ← compiled object files
    mygame        ← linked binary
  dreamcast-kos/
    obj/
    mygame        ← ELF binary
    mygame.cdi    ← CDI image (if cdi target was built)
```

---

## tclmkcfg

A graphical project configurator for creating and editing `.tclproj` files.

```sh
# Create a new project
tclmkcfg

# Open an existing project
tclmkcfg myproject.tclproj

# Start with a specific output filename
tclmkcfg newproject.tclproj
```

### Tabs

**Project** — Name, version, license (default: 0BSD), and description.

**Sources** — Add source file glob patterns (`src/core/*.c`, `src/main.c`)
and exclusions. Use the Browse button to pick files or directories
interactively.

**Toolchains** — Define build platforms. Choose from built-in presets:

- Linux GCC
- Linux Clang
- MinGW-w64 (Windows cross-compile)
- KallistiOS (Sega Dreamcast)
- devkitARM (Game Boy Advance)
- devkitPPC (GameCube / Wii)
- PSn00bSDK (PlayStation 1)
- Custom

Each toolchain has a full set of fields: compiler, flags, defines, include
paths, sysroot, optimisation levels, debug flags, and build targets. Targets
can depend on other targets — for example a `cdi` target that depends on
`binary` will automatically build the binary first.

**Dependencies** — Define git submodule dependencies. Each dependency has a
URL, local path (default: `deps/<name>`), branch/tag/commit, an ordered list
of build commands to compile it, and include/lib paths that `tclmk` will
automatically add as `-I`/`-L`/`-l` flags.

Header-only libraries need no build commands — just set the include path.

**Options** — Feature flags that appear as checkboxes in the `tclmk` GUI.
Each option can be restricted to specific platforms and has a default
on/off state.

**Preview** — Live read-only view of the `.tclproj` file that will be written.
Updates as you make changes in other tabs.

### CMake Import

**File → Import CMakeLists.txt...** attempts a best-effort import of an
existing CMake project. The following CMake commands are recognised:

- `project()` — name and version
- `add_executable()`, `add_library()` — source files
- `target_sources()` — additional source files
- `include_directories()`, `target_include_directories()` — include paths
- `add_compile_definitions()`, `target_compile_definitions()` — defines
- `target_link_libraries()` — link libraries (CMake targets like `SDL2::SDL2`
  are skipped and noted in the results report)
- `set()` — variable assignments used for `${VAR}` expansion
- `find_package()` — skipped with a note (add as a dependency manually)
- `add_subdirectory()` — skipped with a note

A colour-coded results report shows what was imported (✓), skipped (!), and
warned (✗). You can review before committing — Cancel discards the import,
**Apply to Project** populates the configurator and switches to the Preview
tab.

The import creates a `linux-gcc` toolchain populated with the discovered
sources, includes, defines, and link flags. Clone and adjust it for other
target platforms as needed.

### License Fetching

The project license field accepts any
[SPDX identifier](https://spdx.org/licenses/). The license text for any SPDX
ID can be fetched directly from:

```
https://spdx.org/licenses/<SPDX-ID>.txt
```

For example: `https://spdx.org/licenses/0BSD.txt`

---

## .tclproj Format

See `tclproj-spec.md` for the full format specification. A minimal example:

```tcl
project hello {
    version 1.0.0
    license 0BSD
}

sources {
    src src/main.c
}

toolchain linux-gcc {
    set cc  gcc
    set outdir build/linux-gcc
}
```

A more complete example with multiple platforms, dependencies, and targets:

```tcl
project MyGame {
    version     0.1.0
    description "A cross-platform action game"
    license     0BSD
}

sources {
    src     src/core/*.c
    src     src/render/*.c
    src     src/main.c
    exclude src/core/win32_stub.c
}

toolchain linux-gcc {
    set cc             gcc
    set cflags         {-Wall -Wextra}
    set defines        {PLATFORM_LINUX}
    set opt_levels     {-O0 -O1 -O2 -O3 -Os -Oz}
    set default_olevel -O2
    set debug_flags    [dict create -g "GDB symbols"]
    set outdir         build/linux-gcc
}

toolchain dreamcast-kos {
    set cc             kos-cc
    set cflags         {-ml -m4-single-only -Wall}
    set ldflags        {-lkallisti -lc -lgcc}
    set defines        {PLATFORM_DREAMCAST}
    set sysroot        /opt/toolchains/dc/sh-elf
    set outdir         build/dreamcast

    target binary {
        set desc    "Build ELF binary"
        set default 1
    }

    target cdi {
        set desc "Package CDI disc image"
        set deps binary
        set cmds {
            {scramble build/dreamcast/mygame build/dreamcast/1ST_READ.BIN}
            {mkdcdisc -e build/dreamcast/1ST_READ.BIN
                      -o build/dreamcast/mygame.cdi}
        }
    }
}

dependency SDL2 {
    set url        https://github.com/libsdl-org/SDL.git
    set branch     release-2.30.0
    set build_cmds {
        {cmake -B build -DCMAKE_BUILD_TYPE=Release}
        {cmake --build build --parallel}
    }
    set inc_path   include
    set lib_path   build
    set lib_name   SDL2
}

option ENABLE_NETWORKING {
    set desc      "Network support"
    set default   1
    set platforms {linux-gcc dreamcast-kos}
}
```

---

## License

Public domain, but since there are jurisdictions in which the public domain is not legally recognized, it is also made available under the terms of the **BSD 0-Clause (0BSD)** license.
See `LICENSE` for terms.
