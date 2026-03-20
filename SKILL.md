---
name: libfuzzer-oss-packager
description: >
  Use this skill whenever the user wants to compile LibFuzzer harnesses into
  binaries and package them into OSS-Fuzz-compatible zip archives. Triggers on
  phrases like "打包 fuzz harness", "编译 fuzzer", "package fuzzers", "build
  fuzzers", "oss-fuzz package", "生成压缩包", "build_fuzzers", "sanitizer zip",
  "address sanitizer build", "memory sanitizer build", or when the user has
  written harness .c/.cpp files and wants to produce distributable fuzz binaries.
  Also trigger when the user says "模拟 oss-fuzz 产物", "simulate oss-fuzz output",
  or "package for fuzzing platform". Use proactively any time harnesses are ready
  and the user wants to compile or distribute them.
---

# LibFuzzer OSS-Fuzz Packager

Compile all LibFuzzer harnesses for a C/C++ library into sanitizer-instrumented
binaries, then bundle them into OSS-Fuzz-compatible zip packages — one per
sanitizer (AddressSanitizer and MemorySanitizer).

## Final output layout (per sanitizer zip)

```
cjson_libfuzzer_address.zip          ← naming: <libname>_libfuzzer_<sanitizer>.zip
├── <fuzzer_name>                    ← compiled binary (executable)
├── <fuzzer_name>_seed_corpus.zip    ← initial seeds for this fuzzer
├── <fuzzer_name>.dict               ← fuzzer dictionary (if the library provides one)
├── ...                              ← (repeat for each fuzzer)
└── llvm-symbolizer                  ← symbolizer binary for crash reports
```

Both `cjson_libfuzzer_address.zip` and `cjson_libfuzzer_memory.zip` must be
produced in the same run.

---

## Step 1 — Discovery

Before writing any build script, collect these facts about the project:

### 1a. Find harness source files

```bash
find <project_dir> -name "*fuzzer*.c" -o -name "*fuzzer*.cpp" \
                   -o -name "*fuzz*.c"  -o -name "*fuzz*.cpp" | sort
```

Filter out `fuzz_main.c` / `afl.c` — those are driver stubs, not harnesses.
Confirm each file defines `LLVMFuzzerTestOneInput`.

```bash
grep -l "LLVMFuzzerTestOneInput" <candidate_files>
```

### 1b. Check for `build_harness/` and identify the build system

If the project was prepared with `libfuzzer-lib-builder`, a `build_harness/`
directory exists at the project root with pre-built instrumented archives under
`asan/` and `msan/` subdirectories. You can reuse these directly instead of
rebuilding — the cmake invocation is recorded inside the subdirectory:

```bash
ls build_harness/           # asan/  msan/  build_<libname>.sh
ls build_harness/asan/      # lib<name>.a  include/  cmake_build/
# The build script documents the cmake flags used:
head -60 build_harness/build_<libname>.sh
```

If `build_harness/` is absent, the packager must build the library itself
(see the build commands in Step 2).

Whether or not `build_harness/` exists, confirm the build system:

| Evidence | Build system |
|---|---|
| `CMakeLists.txt` at root | CMake |
| `configure` or `configure.ac` | Autoconf |
| `Makefile` (hand-written) | Make |
| `meson.build` | Meson |

### 1c. Find seed directories

Common conventions (check all of them):

| Pattern | Example |
|---|---|
| `seeds_<fuzzer_name>/` | `seeds_cjson_read_fuzzer/` |
| `inputs/` | shared corpus for all fuzzers |
| `corpus/` | shared corpus |
| `<fuzzer_name>_corpus/` | per-fuzzer corpus |

Map each harness → its seed directory. A harness with no seeds simply gets no
`_seed_corpus.zip` (OSS-Fuzz accepts this, but warn the user).

### 1d. Find dictionary files

```bash
find <project_dir> -name "*.dict" -o -name "*.dict.txt" | sort
```

A single dict file (e.g., `json.dict`) usually applies to all fuzzers. Note its path.

### 1e. Confirm output directory

Ask the user where to write the final zips (default: `./out/`).

---

## Step 2 — Write the build script

Generate a shell script (`build_and_package.sh`) tailored to this library.
Use the template below and fill in the library-specific parts.

```bash
#!/bin/bash -eu

# ──────────────────────────────────────────────
# CONFIGURATION — fill these in per-library
# ──────────────────────────────────────────────
LIB_NAME="<libname>"                          # e.g. cjson
SRC="<absolute_path_to_library_source>"       # directory with CMakeLists.txt / Makefile
FUZZING_DIR="<absolute_path_to_fuzzing_dir>"  # directory containing harness .c/.cpp files
FINAL_OUT="<absolute_output_dir>"             # where to write the final zips

CC=clang
CXX=clang++

# All harness base names (no extension, no path)
FUZZERS=(
    fuzzer_one
    fuzzer_two
    # add more
)

# Seed dir for each fuzzer (associative array)
declare -A SEED_DIRS
SEED_DIRS[fuzzer_one]="$FUZZING_DIR/seeds_fuzzer_one"
SEED_DIRS[fuzzer_two]="$FUZZING_DIR/inputs"
# SEED_DIRS[fuzzer_three]=""   ← leave empty string if no seeds exist

# Path to the shared dict file, or "" if none
DICT_FILE="$FUZZING_DIR/library.dict"

# File extension of harness sources ("c" or "cpp")
HARNESS_EXT="c"

# ──────────────────────────────────────────────
# BUILD FUNCTION — called once per sanitizer
# ──────────────────────────────────────────────
build_for_sanitizer() {
    local SANITIZER=$1          # "address" or "memory"
    local OUT_DIR="$(mktemp -d)/build_${SANITIZER}"
    local LIB_BUILD="$(mktemp -d)/lib_${SANITIZER}"
    local PKG_DIR="$FINAL_OUT/pkg_${SANITIZER}"

    mkdir -p "$OUT_DIR" "$LIB_BUILD" "$PKG_DIR"

    # Sanitizer flags
    if [ "$SANITIZER" = "address" ]; then
        CFLAGS="-O1 -fno-omit-frame-pointer -gline-tables-only \
            -DFUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION \
            -fsanitize=address -fsanitize-address-use-after-scope"
        MSAN_CMAKE_FLAGS=""
    else
        # MSan requires ALL inline ASM to be disabled — MSan cannot instrument
        # hand-written assembly that writes to memory, causing false positives.
        # Set MSAN_CMAKE_FLAGS to the library-specific ASM-disable option, e.g.:
        #   libaom:        -DAOM_TARGET_CPU=generic
        #   libjpeg-turbo: -DWITH_SIMD=0
        #   libwebp:       -DWEBP_ENABLE_SIMD=0
        # See libfuzzer-lib-builder references/build_systems.md for the full table.
        CFLAGS="-O1 -fno-omit-frame-pointer -gline-tables-only \
            -DFUZZING_BUILD_MODE_UNSAFE_FOR_PRODUCTION \
            -fsanitize=memory -fsanitize-memory-track-origins"
        MSAN_CMAKE_FLAGS="__REPLACE_WITH_ASM_DISABLE_FLAG__"
    fi
    LIB_FUZZING_ENGINE="-fsanitize=fuzzer"

    echo "=== Building library with -fsanitize=$SANITIZER ==="
    __BUILD_LIBRARY_COMMANDS__     # ← replace with actual build commands (see below)
    # For CMake: append $MSAN_CMAKE_FLAGS to the cmake invocation when SANITIZER=memory

    echo "=== Compiling harnesses ==="
    for FUZZER in "${FUZZERS[@]}"; do
        COMPILER=$CC
        [ "$HARNESS_EXT" = "cpp" ] && COMPILER=$CXX
        $COMPILER $CFLAGS $LIB_FUZZING_ENGINE \
            "$FUZZING_DIR/${FUZZER}.${HARNESS_EXT}" \
            -I"$SRC" \
            -o "$PKG_DIR/$FUZZER" \
            __LINK_FLAGS__         # ← e.g. "$LIB_BUILD/libfoo.a"
        echo "  OK: $FUZZER"
    done

    echo "=== Creating seed corpus zips ==="
    for FUZZER in "${FUZZERS[@]}"; do
        SEED_DIR="${SEED_DIRS[$FUZZER]:-}"
        if [ -n "$SEED_DIR" ] && [ -d "$SEED_DIR" ] && [ -n "$(ls -A "$SEED_DIR" 2>/dev/null)" ]; then
            find "$SEED_DIR" -maxdepth 1 -type f | \
                xargs zip -j "$PKG_DIR/${FUZZER}_seed_corpus.zip" > /dev/null
            echo "  OK: ${FUZZER}_seed_corpus.zip"
        else
            echo "  SKIP: no seeds for $FUZZER"
        fi
    done

    echo "=== Copying dict and llvm-symbolizer ==="
    if [ -n "$DICT_FILE" ] && [ -f "$DICT_FILE" ]; then
        for FUZZER in "${FUZZERS[@]}"; do
            cp "$DICT_FILE" "$PKG_DIR/${FUZZER}.dict"
        done
        echo "  Copied dict for all fuzzers"
    fi
    cp "$(which llvm-symbolizer)" "$PKG_DIR/llvm-symbolizer"

    echo "=== Packaging ==="
    ZIP="$FINAL_OUT/${LIB_NAME}_libfuzzer_${SANITIZER}.zip"
    cd "$PKG_DIR" && zip -r "$ZIP" . && cd -
    echo "Done: $ZIP"
}

mkdir -p "$FINAL_OUT"
build_for_sanitizer "address"
build_for_sanitizer "memory"

echo ""
echo "All packages:"
ls -lh "$FINAL_OUT"/*.zip
```

### Reusing pre-built archives from `libfuzzer-lib-builder`

If `build_harness/asan/` and `build_harness/msan/` already exist, skip the library
rebuild inside `build_for_sanitizer` and link directly against the pre-built archive.
Replace `__BUILD_LIBRARY_COMMANDS__` and `__LINK_FLAGS__` with:

```bash
# Reuse pre-built archives — no rebuild needed
if [ "$SANITIZER" = "address" ]; then
    PREBUILT="$PROJECT_ROOT/build_harness/asan"
else
    PREBUILT="$PROJECT_ROOT/build_harness/msan"
fi
# Then in the harness compile loop, add -I"$PREBUILT/include" and link "$PREBUILT/lib<name>.a"
```

### Build library commands per build system

Fill in `__BUILD_LIBRARY_COMMANDS__` based on the build system found in Step 1
(only needed when `build_harness/` was NOT created by `libfuzzer-lib-builder`):

**CMake:**
```bash
cd "$LIB_BUILD"
cmake "$SRC" \
    -DBUILD_SHARED_LIBS=OFF \
    -DENABLE_TESTING=OFF \          # disable tests to avoid linking issues (flag varies per library)
    -DCMAKE_C_COMPILER=$CC \
    -DCMAKE_C_FLAGS="$CFLAGS" \
    -DCMAKE_BUILD_TYPE=Release \
    ${MSAN_CMAKE_FLAGS:+$MSAN_CMAKE_FLAGS} \   # MSan only: disables inline ASM
    -Wno-dev > cmake.log 2>&1
make -j$(nproc) >> cmake.log 2>&1
# Static lib is usually at: $LIB_BUILD/lib<name>.a
```

**Make (in-tree, with CFLAGS override):**
```bash
cd "$SRC"
make -j$(nproc) CC=$CC CFLAGS="$CFLAGS" lib<name>.a
```

**Autoconf:**
```bash
cd "$LIB_BUILD"
"$SRC/configure" --disable-shared --enable-static CC=$CC CFLAGS="$CFLAGS"
make -j$(nproc)
```

Fill in `__LINK_FLAGS__` with the path to the static library, e.g.:
```bash
"$LIB_BUILD/libcjson.a"
```

---

## Step 3 — Critical compiler choice for C harnesses

**Always use `clang` (not `clang++`) to compile `.c` harnesses.**

When `clang++` compiles a `.c` file, C++ name mangling applies to
`LLVMFuzzerTestOneInput`, causing an "undefined reference" link error even though
the function is clearly defined. Using `clang` avoids this entirely.

For `.cpp` harnesses, use `clang++` as normal (C++ linkage is expected).

The template above handles this automatically via the `HARNESS_EXT` / `COMPILER`
logic.

---

## Step 4 — Run the script and verify

```bash
bash build_and_package.sh 2>&1 | tee build.log
```

After it completes, verify the checklist:

```
For each sanitizer zip:
  ✓ <fuzzer>                 binary exists and is executable
  ✓ <fuzzer>_seed_corpus.zip exists (if seeds were present)
  ✓ <fuzzer>.dict            exists (if a dict file was provided)
  ✓ llvm-symbolizer          exists
```

Quick check:
```bash
for ZIP in out/*.zip; do
    echo "=== $ZIP ==="
    unzip -l "$ZIP" | awk '{print $NF}' | grep -v "^$\|^Name\|^---\|files$" | sort
done
```

If a binary is missing, check `build.log` for the specific compilation error and fix.

---

## Step 5 — Clean up

Remove temporary build directories (`build_tmp/`, intermediate `.a` files, the
`build_and_package.sh` script itself) after the zips are confirmed correct. Only
`out/` with the two zips should remain.

```bash
rm -rf build_tmp/ build_and_package.sh
```

---

## Common failure modes and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `undefined reference to LLVMFuzzerTestOneInput` | Compiled `.c` with `clang++` | Use `clang` for C harnesses |
| `cannot find -lasan` / `-lmsan` | clang not installed or wrong version | Install `clang` / `llvm` via apt |
| MSan build fails or produces many false positives from libaom / libjpeg etc. | Inline ASM not disabled for MSan | Set `MSAN_CMAKE_FLAGS` to the library's ASM-disable option (e.g., `-DAOM_TARGET_CPU=generic`, `-DWITH_SIMD=0`) — see `libfuzzer-lib-builder` references/build_systems.md |
| MSAN: false positives from uninstrumented libc | libc not MSAN-instrumented | Expected outside OSS-Fuzz Docker; real bugs show user-code frames in the origin stack |
| Shared lib linked instead of static | `-DBUILD_SHARED_LIBS` not set to OFF | Add `-DBUILD_SHARED_LIBS=OFF` to cmake |
| Empty seed zip | Wrong seed directory path | Re-run Step 1c to verify paths |
| `llvm-symbolizer: not found` | llvm not installed | `apt install llvm` |

---

## Naming conventions reference

| Artifact | Naming rule |
|---|---|
| Final zip | `<libname>_libfuzzer_<sanitizer>.zip` |
| Seed corpus zip | `<fuzzer_name>_seed_corpus.zip` |
| Dictionary | `<fuzzer_name>.dict` |
| Binary | `<fuzzer_name>` (no extension) |

The `<libname>` should be lowercase, matching the OSS-Fuzz project name (e.g.,
`cjson`, `libpng`, `sqlite3`).
