# `.tclproj` File Format Specification

**Version:** 1.0  
**Tools:** tclmk, tclmkcfg

---

## Overview

A `.tclproj` file is a Tcl script evaluated directly by `tclmk` to describe a
software project. Because it is plain Tcl, it supports variables, conditionals,
loops, and any other Tcl construct — but for maximum compatibility with
`tclmkcfg` and other tooling, project files should be written using only the
DSL commands described in this document unless the author explicitly accepts
reduced tool support.

The default filename is `build.tclproj`. An alternate path may be specified
with `tclmk --project <file>`.

---

## Encoding and Syntax

- Files must be UTF-8 encoded.
- Line endings may be LF or CRLF; LF is preferred.
- All DSL commands use Tcl brace-delimited blocks `{ }` for their bodies.
- String values containing spaces must be quoted or braced: `{my value}` or
  `"my value"`.
- List values are space-separated Tcl lists: `{-O2 -Wall -Wextra}`.

---

## Top-Level DSL Commands

The following commands are recognised at the top level of a `.tclproj` file.
All are optional except `project`, which should appear first.

---

### `project`

Declares project metadata.

```tcl
project <name> {
    version     <string>
    description <string>
    license     <string>
}
```

**Fields:**

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes (as argument) | Project identifier, no spaces |
| `version` | string | no | Semantic version string, e.g. `1.0.0` |
| `description` | string | no | Short human-readable description |
| `license` | string | no | SPDX license identifier, e.g. `0BSD` (default: `0BSD`) |

**Example:**

```tcl
project MyGame {
    version     0.3.1
    description "A cross-platform action game"
    license     0BSD
}
```

---

### `sources`

Declares the source files to compile. The `src` sub-command accepts any glob
pattern — use a specific filename for a single file, or a glob pattern to match
multiple files. All patterns are expanded relative to the project root.

```tcl
sources {
    src     <glob-pattern>
    exclude <glob-pattern>
}
```

**Sub-commands:**

| Command | Argument | Description |
|---|---|---|
| `src` | glob pattern | Include all files matching the pattern |
| `exclude` | glob pattern | Exclude matching files from the set |

All paths are relative to the directory from which `tclmk` is invoked (the
project root). Multiple `src` and `exclude` entries may appear in any order.
Exclusions are applied after all inclusions are collected.

**Example:**

```tcl
sources {
    src     src/core/*.c
    src     src/core/*.cpp
    src     src/panel/*.c
    src     src/main.c
    exclude src/core/win32_stub.c
    exclude src/core/*_test.c
}
```

---

### `toolchain`

Declares a named build toolchain (platform target). Multiple `toolchain` blocks
may appear; each defines one buildable platform.

```tcl
toolchain <name> {
    set cc             <string>
    set cxx            <string>
    set ar             <string>
    set cflags         <list>
    set cxxflags       <list>
    set ldflags        <list>
    set defines        <list>
    set includes       <list>
    set sysroot        <string>
    set opt_levels     <list>
    set default_olevel <string>
    set debug_flags    [dict create <flag> <desc> ...]
    set outdir         <string>
}
```

**Fields:**

| Field | Type | Default | Description |
|---|---|---|---|
| `cc` | string | `gcc` | C compiler executable |
| `cxx` | string | `g++` | C++ compiler executable |
| `ar` | string | `ar` | Archiver executable |
| `cflags` | list | `{}` | C compiler flags |
| `cxxflags` | list | `{}` | C++ compiler flags |
| `ldflags` | list | `{}` | Linker flags |
| `defines` | list | `{}` | Preprocessor defines (without `-D`) |
| `includes` | list | `{}` | Include paths (without `-I`) |
| `sysroot` | string | `""` | Sysroot path, passed as `--sysroot=` |
| `opt_levels` | list | `{-O0 -O1 -O2 -O3 -Os}` | Available optimisation levels shown in GUI |
| `default_olevel` | string | `-O2` | Default optimisation level |
| `debug_flags` | dict | `{}` | Available debug flags: keys are flags, values are descriptions |
| `outdir` | string | `build/<name>` | Output directory for objects and final binary |


Fields are set using Tcl `set` inside the body, not as sub-commands. This is
intentional: the body is evaluated as a namespace, so standard Tcl assignment
applies.

#### `target` (nested inside `toolchain`)

Declares a named build target within the toolchain. The `binary` target is
always implicitly defined and represents the compile+link step. Additional
targets run arbitrary shell commands and may depend on other targets.

`tclmk` resolves the full dependency chain automatically — requesting `cdi`
will build `binary` first if `cdi` declares `deps binary`, and so on
transitively.

**Target fields:**

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | yes (as argument) | Target identifier |
| `desc` | string | `""` | Human-readable description |
| `deps` | list | `{}` | Names of targets that must be built before this one |
| `cmds` | list | `{}` | Ordered shell commands to execute for this target |
| `default` | 0 or 1 | `0` | Whether this target is built by default (`binary` defaults to `1`) |

**CLI target syntax:**

```bash
tclmk --nogui dreamcast-kos:cdi -O2
tclmk --nogui dreamcast-kos:binary dreamcast-kos:cdi -O2
```

**Example:**

```tcl
toolchain linux-gcc {
    set cc             gcc
    set cxx            g++
    set ar             ar
    set cflags         {-Wall -Wextra}
    set ldflags        {}
    set defines        {PLATFORM_LINUX}
    set opt_levels     {-O0 -O1 -O2 -O3 -Os -Oz}
    set default_olevel -O2
    set debug_flags    [dict create \
        -g                 "GDB symbols" \
        -fsanitize=address "AddressSanitizer" \
        -pg                "gprof profiling"]
    set outdir         build/linux-gcc
}

toolchain dreamcast-kos {
    set cc             kos-cc
    set cxx            kos-c++
    set ar             kos-ar
    set cflags         {-ml -m4-single-only -Wall}
    set ldflags        {-lkallisti -lc -lgcc}
    set defines        {PLATFORM_DREAMCAST _arch_dreamcast}
    set sysroot        /opt/toolchains/dc/sh-elf
    set opt_levels     {-O0 -O1 -O2 -Os}
    set default_olevel -O2
    set debug_flags    [dict create \
        -g              "GDB stub symbols" \
        FRAME_POINTERS  "KOS frame pointers" \
        DC_SERIAL_DEBUG "Serial port debug output"]
    set outdir         build/dreamcast

    target binary {
        set desc    "Build ELF binary"
        set default 1
    }

    target cdi {
        set desc    "Package CDI disc image"
        set deps    binary
        set cmds    {
            {makeip IP.TMPL build/dreamcast/IP.BIN}
            {scramble build/dreamcast/mygame build/dreamcast/1ST_READ.BIN}
            {mkdcdisc -e build/dreamcast/1ST_READ.BIN
                      -i build/dreamcast/IP.BIN
                      -o build/dreamcast/mygame.cdi}
        }
    }
}
```


---

### `dependency`

Declares a git submodule dependency. Multiple `dependency` blocks may appear.
`tclmk` will automatically initialise missing submodules and build unbuilt
dependencies before building any target.

```tcl
dependency <name> {
    set path       <string>
    set url        <string>
    set branch     <string>
    set build_cmds <list>
    set inc_path   <string>
    set lib_path   <string>
    set lib_name   <string>
}
```

**Fields:**

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | yes (as argument) | Identifier for this dependency |
| `path` | string | `deps/<name>` | Local path relative to project root |
| `url` | string | `""` | Git repository URL for submodule |
| `branch` | string | `main` | Branch, tag, or commit to check out |
| `build_cmds` | list | `{}` | Ordered shell commands to build the dependency, run inside `path` |
| `inc_path` | string | `""` | Include path relative to `path` root, added as `-I`. If blank, `path` itself is used as the include root |
| `lib_path` | string | `""` | Library output path relative to `path` root, added as `-L` |
| `lib_name` | string | `""` | Library name without `lib` prefix, added as `-l` |

**Behaviour:**

- If `url` is non-empty and `path` is missing or empty, `tclmk` runs
  `git submodule update --init --recursive <path>` before building.
- If `build_cmds` is empty, the dependency is treated as header-only: the
  submodule is initialised if needed but no build step is run.
- After a successful build, `tclmk` writes a `.tclmk_built` sentinel file
  inside `path`. On subsequent builds, the dependency is skipped unless the
  sentinel is absent (e.g. after `--pull` or manual removal).
- `inc_path` and `lib_path` are relative to `path`, not the project root.

**Examples:**

```tcl
# Library with a configure + make build
dependency zlib {
    set url        https://github.com/madler/zlib.git
    set branch     v1.3.1
    set build_cmds {./configure {make -j4}}
    set inc_path   ""
    set lib_path   ""
    set lib_name   z
}

# Header-only library
dependency stb {
    set url      https://github.com/nothings/stb.git
    set branch   master
    set inc_path ""
}

# CMake-based library
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
```

---

### `option`

Declares a user-selectable feature flag. Options appear as checkboxes in the
`tclmk` GUI and may be passed as `-D` flags on the command line. Multiple
`option` blocks may appear.

```tcl
option <name> {
    set desc      <string>
    set default   <0|1>
    set platforms <list>
}
```

**Fields:**

| Field | Type | Default | Description |
|---|---|---|---|
| `name` | string | yes (as argument) | Preprocessor define name, e.g. `ENABLE_FOO` |
| `desc` | string | `""` | Human-readable description shown in GUI |
| `default` | 0 or 1 | `0` | Whether the option is enabled by default |
| `platforms` | list | `{}` | Toolchain names this option applies to. Empty means all platforms |

When an option is enabled, `tclmk` adds `-D<name>` to the compiler flags for
applicable platforms.

**Example:**

```tcl
option ENABLE_NETWORKING {
    set desc      "Network support"
    set default   1
    set platforms {linux-gcc dreamcast-kos}
}

option ENABLE_ASAN {
    set desc      "AddressSanitizer (debug builds only)"
    set default   0
    set platforms {linux-gcc}
}
```

---

## Sentinel Files

`tclmk` writes the following files during operation. These should be added to
`.gitignore`:

| File | Location | Purpose |
|---|---|---|
| `.tclmk_built` | `<dep path>/` | Marks a dependency as successfully built |

---

## Build Output Layout

By default, build output is written to `build/<toolchain-name>/` relative to
the project root:

```
<project root>/
    build/
        linux-gcc/
            obj/        <- compiled object files
            <name>      <- linked binary
        dreamcast-kos/
            obj/
            <name>
    deps/
        <dep-name>/     <- git submodule contents
            .tclmk_built
```

The output directory for each toolchain may be overridden via the `outdir`
field in the `toolchain` block.

---

## Compatibility Notes

- `.tclproj` files require **Tcl 8.6** or later.
- The `dict create` construct used in `debug_flags` requires Tcl 8.5+; 8.6 is
  the minimum supported version regardless.
- Tools consuming `.tclproj` files must tolerate unknown top-level commands
  gracefully (ignore them) to allow forward compatibility with future DSL
  additions.
- Fields within a block that are unrecognised by the consuming tool should
  likewise be ignored.
- The order of top-level blocks is not significant except that `project` should
  appear first by convention.

---

## Minimal Valid Example

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
    set out build/linux-gcc
}
```

---

## Full Example

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
    set cxx            g++
    set ar             ar
    set cflags         {-Wall -Wextra}
    set defines        {PLATFORM_LINUX}
    set opt_levels     {-O0 -O1 -O2 -O3 -Os -Oz}
    set default_olevel -O2
    set debug_flags    [dict create \
        -g                 "GDB symbols" \
        -fsanitize=address "AddressSanitizer"]
    set outdir         build/linux-gcc
}

toolchain dreamcast-kos {
    set cc             kos-cc
    set cxx            kos-c++
    set cflags         {-ml -m4-single-only -Wall}
    set ldflags        {-lkallisti -lc -lgcc}
    set defines        {PLATFORM_DREAMCAST _arch_dreamcast}
    set sysroot        /opt/toolchains/dc/sh-elf
    set opt_levels     {-O0 -O1 -O2 -Os}
    set default_olevel -O2
    set debug_flags    [dict create \
        -g              "GDB stub symbols" \
        DC_SERIAL_DEBUG "Serial port debug output"]
    set outdir         build/dreamcast
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

dependency stb {
    set url      https://github.com/nothings/stb.git
    set branch   master
    set inc_path ""
}

option ENABLE_NETWORKING {
    set desc      "Network support"
    set default   1
    set platforms {linux-gcc dreamcast-kos}
}

option ENABLE_ASAN {
    set desc      "AddressSanitizer"
    set default   0
    set platforms {linux-gcc}
}
```
