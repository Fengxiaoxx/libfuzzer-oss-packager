# libfuzzer-oss-packager

A Claude Code skill that compiles LibFuzzer harnesses for any C/C++ library and
packages them into OSS-Fuzz-compatible zip archives — one per sanitizer.

## What it does

Given a C/C++ library with one or more LibFuzzer harness files, this skill guides
Claude through:

1. **Discovery** — locating harness `.c/.cpp` files, seed directories, dictionary
   files, and identifying the build system (CMake / Make / Autoconf / Meson)
2. **Compilation** — building the library and every harness with both
   AddressSanitizer (`-fsanitize=address`) and MemorySanitizer (`-fsanitize=memory`),
   linked against the libFuzzer runtime (`-fsanitize=fuzzer`)
3. **Corpus packaging** — zipping each fuzzer's seed corpus into
   `<fuzzer>_seed_corpus.zip`
4. **Final packaging** — assembling all artifacts into two ready-to-deploy archives

## Output layout

```
out/
├── <libname>_libfuzzer_address.zip
│   ├── <fuzzer_name>                 ← ASAN-instrumented binary
│   ├── <fuzzer_name>_seed_corpus.zip ← initial seeds
│   ├── <fuzzer_name>.dict            ← fuzzer dictionary (if provided)
│   └── llvm-symbolizer
└── <libname>_libfuzzer_memory.zip
    ├── <fuzzer_name>                 ← MSAN-instrumented binary
    ├── <fuzzer_name>_seed_corpus.zip
    ├── <fuzzer_name>.dict
    └── llvm-symbolizer
```

This layout matches the output of `python3 infra/helper.py build_fuzzers` in a
real OSS-Fuzz environment, making the archives directly usable with OSS-Fuzz
tooling or any fuzzing platform that follows the same convention.

## Installation

### Option A — Install from `.skill` file

Download `libfuzzer-oss-packager.skill` from this repository, then run:

```bash
claude skill install libfuzzer-oss-packager.skill
```

### Option B — Manual install

```bash
# Clone this repo into your Claude skills directory
git clone https://github.com/Fengxiaoxx/libfuzzer-oss-packager.git \
    ~/.claude/skills/libfuzzer-oss-packager
```

Restart Claude Code. The skill will appear in the available skills list automatically.

## Usage

Once installed, just describe what you want in natural language:

```
我写好了 libpng 的 harness，帮我打包成 oss-fuzz 格式的压缩包
```

```
package my libxml2 fuzzers for both address and memory sanitizers, output to ./out
```

```
build_fuzzers for sqlite3, I have 3 harness files in src/fuzzing/
```

Claude will invoke this skill automatically and walk through the discovery →
build → package → verify workflow.

## Compatibility

| Build system | Supported |
|---|---|
| CMake | ✅ |
| Make | ✅ |
| Autoconf | ✅ |
| Meson | ✅ |

| Harness language | Supported |
|---|---|
| C (`.c`) | ✅ (compiled with `clang`) |
| C++ (`.cpp`) | ✅ (compiled with `clang++`) |

**Requirement:** `clang` / `llvm` must be installed on the host (LLVM 14+ recommended).

```bash
sudo apt install clang llvm
```

## Key technical notes

- **C harnesses must be compiled with `clang`, not `clang++`.**
  Using `clang++` causes C++ name mangling on `LLVMFuzzerTestOneInput`, which
  produces an "undefined reference" link error even though the function is
  clearly defined. The skill handles this automatically.

- **Static libraries only.**
  The library is always built as a static archive (`.a`) so the fuzzer binary is
  fully self-contained. Shared libraries (`.so`) are excluded from the link.

- **MSAN outside OSS-Fuzz Docker.**
  MemorySanitizer requires an instrumented libc to avoid false positives. The
  binaries produced outside the OSS-Fuzz Docker image will function correctly
  for coverage-driven fuzzing but may report spurious MSAN errors from
  uninstrumented system libraries. This is expected behavior.

## Repository contents

| File | Description |
|---|---|
| `SKILL.md` | Skill instructions loaded into Claude's context |
| `libfuzzer-oss-packager.skill` | Packaged skill archive (install with `claude skill install`) |
| `README.md` | This file |

## Related skills

- [`libfuzzer-harness-writer`](https://github.com/google/oss-fuzz) — write new
  LibFuzzer harnesses for uncovered API surface
- [`libfuzzer-seed-generator`](https://github.com/google/oss-fuzz) — generate
  high-quality initial seed corpora for harnesses

## License

MIT
