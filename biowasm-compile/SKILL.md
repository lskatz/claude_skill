---
name: biowasm-compile
description: Compile bioinformatics C/C++ tools to WebAssembly using Emscripten following biowasm patterns. Use when compiling genomics tools to WASM, porting tools to the browser, creating compile.sh scripts for tools like SKESA/Mash/minimap2/samtools, building WebAssembly versions of C/C++ scientific tools, or when the user mentions biowasm, emscripten, wasm32, or browser-based genomics pipelines.
argument-hint: "[tool-name]"
allowed-tools: Bash(emcc *), Bash(em++ *), Bash(emcmake *), Bash(emmake *), Bash(docker *)
---

# Biowasm Compile Skill

A guide for compiling bioinformatics C/C++ tools to WebAssembly using
Emscripten, following the patterns established by the biowasm project.

## Overview

The biowasm project (github.com/biowasm/biowasm) has established a reliable
pattern for compiling genomics tools to WASM. Each tool gets a `compile.sh`
that:
1. Downloads the tool source
2. Compiles dependencies (if needed)
3. Compiles the tool with `emcc`/`em++`
4. Outputs `.wasm` + `.js` glue

## Environment

All biowasm compilation happens inside a Docker container based on
`emscripten/emsdk`. This ensures reproducibility and avoids local emsdk
version conflicts.

```dockerfile
FROM emscripten/emsdk:3.1.50
RUN apt-get update && apt-get install -y \
    cmake autoconf automake libtool \
    zlib1g-dev libbz2-dev
```

The key emscripten version used by biowasm is pinned — always check the
biowasm repo for the current pinned version before starting.

### Docker-based compilation (recommended)

Create a `compile-docker.sh` that mounts the project and runs compilation
inside the Docker container:

```bash
#!/bin/bash
set -e
docker run --rm --platform linux/amd64 \
  -v "$(pwd):/work" \
  emscripten/emsdk:3.1.50 \
  bash /work/compile.sh
```

And a `compile.sh` that runs inside the container with access to `/work`.
This approach avoids installing Emscripten locally and ensures reproducible
builds across different host platforms (including Apple Silicon via
`--platform linux/amd64`).

---

## compile.sh Structure

Every biowasm tool follows this shell script pattern:

```bash
#!/bin/bash
# ============================================================
# Compile <TOOLNAME> to WebAssembly
# ============================================================
set -e

# --- Config ---
TOOL_VERSION="X.Y.Z"
TOOL_URL="https://github.com/org/tool/archive/refs/tags/vX.Y.Z.tar.gz"

# --- Download ---
wget -q "$TOOL_URL" -O tool.tar.gz
tar -xzf tool.tar.gz
cd tool-${TOOL_VERSION}

# --- Dependencies (if any) ---
# See references/dependencies.md for common dep compile patterns

# --- Compile ---
emcmake cmake . \
  -DCMAKE_BUILD_TYPE=Release \
  [tool-specific flags]

emmake make -j$(nproc)

# --- Link ---
em++ ${OBJECTS} \
  -O3 \
  -s WASM=1 \
  -s ALLOW_MEMORY_GROWTH=1 \
  -s MAXIMUM_MEMORY=4GB \
  -s EXPORTED_FUNCTIONS="['_main']" \
  -s EXPORTED_RUNTIME_METHODS="['callMain','FS']" \
  -s ENVIRONMENT='web,worker' \
  -o /out/tool.js
```

---

## Common Emscripten Flags Reference

### Memory
```
-s ALLOW_MEMORY_GROWTH=1     # Required for genomics tools — inputs are large
-s MAXIMUM_MEMORY=4GB        # Max for wasm32; use 16GB with MEMORY64
-s INITIAL_MEMORY=64MB       # Starting heap; tune per tool
```

### Exports
```
-s EXPORTED_FUNCTIONS="['_main']"
-s EXPORTED_RUNTIME_METHODS="['callMain','FS','getValue','setValue']"
```
`callMain` lets JS invoke the tool as if running from CLI.
`FS` exposes the Emscripten virtual filesystem for reading/writing files.

### Environment
```
-s ENVIRONMENT='web,worker'   # Browser only; drop 'node' for smaller build
-s MODULARIZE=1               # Wraps output in a factory function (recommended)
-s EXPORT_NAME='createTool'   # Name of the factory function
```

### Threading
```
-s USE_PTHREADS=1             # Enable if tool uses threads (requires COOP/COEP headers)
-s PTHREAD_POOL_SIZE=4        # Pre-spawn N threads
```
⚠️ Threading requires server headers:
`Cross-Origin-Opener-Policy: same-origin`
`Cross-Origin-Embedder-Policy: require-corp`

### Dependencies
```
-s USE_BOOST_HEADERS=1        # Boost header-only libs — no separate compile needed
-s USE_ZLIB=1                 # zlib port built into emscripten
-s USE_BZIP2=1                # bzip2 port
```

---

## Tool-Specific Patterns

### Makefile-based tools (e.g. SKESA, samtools)

For tools with a `Makefile` rather than CMake:

```bash
# Replace compiler in Makefile
emmake make -f Makefile.nongs \
  CXX=em++ \
  CC=emcc \
  CXXFLAGS="-O3 -s USE_BOOST_HEADERS=1" \
  LDFLAGS="-s WASM=1 -s ALLOW_MEMORY_GROWTH=1"
```

For SKESA specifically — use `Makefile.nongs` to skip the SRA/NGS dependency.
SKESA's threading can be disabled by patching `--cores 1` as default, or
stripping the omp/parallel sections for a simpler build.

### CMake-based tools (e.g. Mash, minimap2)

```bash
emcmake cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CXX_FLAGS="-O3" \
  -DBUILD_SHARED_LIBS=OFF

emmake make -j$(nproc)
```

### Rust tools (e.g. sparrowhawk)

Use `wasm-pack` instead of emscripten:

```bash
# In Cargo.toml ensure: [lib] crate-type = ["cdylib"]
wasm-pack build --target web --release -- -F wasm
```

For `wasm32-unknown-unknown` targets with no filesystem:
- Input/output must go through JS/WASM memory boundary
- Use `wasm_bindgen` for the JS interface
- See sparrowhawk-web for a working example

---

## Filesystem and I/O

WASM tools use Emscripten's virtual filesystem (MEMFS by default).

**Preloading files at build time** (small reference files):
```bash
--preload-file reference.fa
```

**Writing files from JS at runtime** (user uploads):
```javascript
const module = await createTool({
  noInitialRun: true,  // Don't run main() on load
  print: (t) => stdout.push(t),     // Capture stdout
  printErr: (t) => stderr.push(t),  // Capture stderr
});
// Write input to virtual FS
module.FS.writeFile('/input.fastq', new Uint8Array(fileBuffer));
// Run tool via callMain (preferred — works like CLI)
try {
  module.callMain(['--reads', '/input.fastq', '--output', '/out.fa']);
} catch (err) { /* Some tools throw on exit */ }
// Read output from virtual FS or stdout array
const result = module.FS.readFile('/out.fa', { encoding: 'utf8' });
```

**Key integration tips:**
- Use `noInitialRun: true` + `callMain()` rather than custom exported
  functions. This is simpler and preserves the tool's argument parsing.
- Capture stdout/stderr via `print`/`printErr` callbacks — many tools write
  output to stdout (FASTA) and diagnostics to stderr.
- In Node.js, pass `wasmBinary: readFileSync('tool.wasm')` to avoid fetch.
- Some tools call `exit()` which throws in WASM — wrap `callMain` in try/catch.

**Streaming large files** — for files >100MB, use WORKERFS or chunk the input
rather than loading into MEMFS all at once.

---

## Output Files

A successful biowasm compile produces:
```
tool.js      # Emscripten glue — load this in the browser
tool.wasm    # The compiled binary — fetched automatically by tool.js
tool.data    # Preloaded filesystem data (only if --preload-file used)
```

Host both `tool.js` and `tool.wasm` from the same directory. The JS file
contains a hardcoded relative path to the WASM file.

---

## Integration with Aioli (biowasm runtime)

biowasm tools are designed to run via the **Aioli** library, which handles:
- Loading WASM modules in a Web Worker
- Shared filesystem between multiple tools
- CLI-style `exec()` calls from JS

```javascript
import { Aioli } from "@biowasm/aioli";

const CLI = await new Aioli(["toolname/version"]);
await CLI.fs.writeFile("/input.fastq", fileContents);
const output = await CLI.exec("toolname --reads /input.fastq");
```

If integrating into a non-Aioli project (like GenomicX), you load the
emscripten module directly — see the Filesystem section above.

---

## Debugging Compilation Failures

**Boost linking errors** — check if you only need headers (`-s USE_BOOST_HEADERS=1`) vs compiled libs. If the tool needs compiled Boost libs (program_options, iostreams, system), build them with Emscripten's `toolset=gcc-emscripten` (see `references/dependencies.md`).

**`unknown argument` linker errors** — emscripten's `wasm-ld` doesn't support all gcc linker flags. Common culprits: `-Bstatic`, `-Bdynamic`, `-rdynamic`. Strip these from `LDFLAGS`.

**`__aarch64__` / architecture ifdefs** — WASM is its own arch. Guard blocks with `#ifndef __EMSCRIPTEN__` or patch the source.

**Threading compile errors** — if the tool uses `<thread>` or OpenMP without `-s USE_PTHREADS=1`, either add the flag or disable threading in source. For single-threaded WASM builds, add `#ifdef __EMSCRIPTEN__` guards to force `ncores=1`.

**C++ exceptions** — many bioinformatics tools use exceptions. You MUST add both `-fexceptions` to CXXFLAGS and `-s DISABLE_EXCEPTION_CATCHING=0` to link flags. Without this, any `throw` will call `abort()`.

**Binary too large** — add `-Os` instead of `-O3`, and `--closure 1` for JS minification. Strip unused exports.

---

## Critical: wasm32 Portability Bugs

**This is the #1 source of silent runtime bugs.** A tool may compile and link
successfully but produce wrong results because of 32-bit vs 64-bit differences.

On wasm32, `size_t` and `ptrdiff_t` are **32-bit** (vs 64-bit on x86-64).
This causes silent data corruption in any code that:

### 1. Left-shifts by >= 32 bits on `size_t` or `ptrdiff_t`

```cpp
// BUG: ptrdiff_t is 32-bit on wasm32, shift >= 32 is UB
storage[word] += ((find(table.begin(), table.end(), ch) - table.begin()) << shift);
// FIX: cast to uint64_t before shifting
storage[word] += (uint64_t(find(table.begin(), table.end(), ch) - table.begin()) << shift);
```

WASM's `i32.shl` masks the shift to 5 bits: `<< 32` becomes `<< 0`,
`<< 33` becomes `<< 1`, etc. This silently corrupts data.

### 2. Packs data into upper bits of `size_t`

```cpp
// BUG: size_t is 32-bit on wasm32, shifts << 48 and << 32 overflow
size_t packed = (strand << 48) + (branch << 32) + count;
// FIX: use uint64_t explicitly
uint64_t packed = (uint64_t(strand) << 48) + (uint64_t(branch) << 32) + count;
```

Common in genomics tools that pack metadata (strand info, branching, flags)
into the upper bits of count values.

### 3. Stores hash values in `size_t`

```cpp
// BUG: 64-bit hash truncated to 32-bit on wasm32
size_t hash_val = kmer.hash64();  // returns uint64_t, truncated to 32-bit
size_t bucket = hash_val % table_size;  // different distribution than native
// FIX: keep hash as uint64_t until modulo
uint64_t hash_val = kmer.hash64();
size_t bucket = hash_val % table_size;
```

### How to find these bugs

After compilation succeeds, if the tool produces **wrong results** but no
crashes:

1. **Search for `size_t` in bit-shift operations:** `grep -n 'size_t.*<<\|<<.*size_t'`
2. **Search for shifts >= 32:** `grep -n '<< 32\|<< 48\|<< 40'`
3. **Search for bit-packing patterns:** `grep -n 'upper.*bit\|pack.*count\|strand.*shift'`
4. **Search for `ptrdiff_t` or iterator arithmetic in shifts:** Look for
   `(find(...) - begin()) << shift` patterns
5. **Compare output with native build** — if counts, hashes, or statistics
   differ, suspect `size_t` truncation
6. **Add `#ifdef __EMSCRIPTEN__` debug output** at key pipeline stages to
   narrow down where values diverge

### The fix pattern

Change `size_t` to `uint64_t` in:
- Pair types storing packed data: `pair<Kmer, size_t>` → `pair<Kmer, uint64_t>`
- All functions that read/write the packed data (getters, setters, visitors)
- Local variables involved in bit-shifting before the result is stored
- Cast arithmetic results to `uint64_t` before shifting left by >= 32

---

## Tool-Specific Patterns

Different bioinformatics tools have different compile quirks. **Always check `references/tool-specific.md` before starting** - it contains real-world patterns from 40+ tools in the biowasm project.

### Quick Pattern Guide

**Simple tools (seqtk, etc.):**
```bash
emcc tool.c -o tool.js -O2 $EM_FLAGS
```

**Autoconf tools (samtools, bcftools, htslib):**
```bash
autoheader && autoconf
emconfigure ./configure --with-htslib=path
emmake make CC=emcc AR=emar LDFLAGS="-s ERROR_ON_UNDEFINED_SYMBOLS=0"
```

**SIMD-optimized (minimap2):**
- Compile both SIMD (`-msimd128`) and non-SIMD (`Makefile.simde`) versions
- SIMD is 2-3x faster but non-SIMD ensures compatibility

**Tools with x86 assembly (bowtie2):**
- Disable architecture-specific features: `POPCNT_CAPABILITY=0`, `NO_TBB=1`
- Look for `#ifdef __x86_64__` blocks in source

**Tools bundling zlib (fastp):**
- Use `-DDYNAMIC_ZLIB -s USE_ZLIB=1` to avoid function signature mismatches
- Prevents crashes on `.gz` files

**Tools needing async JS (bhtsne):**
- Use `-s ASYNCIFY=1 -s ASYNCIFY_IMPORTS=["func1","func2"]`
- Required for progress callbacks or async API calls
- Adds ~30-50KB to binary

**See `references/tool-specific.md` for detailed patterns and debugging tips.**

---

## Additional Resources

For detailed reference material, see:

- **[references/tool-specific.md](references/tool-specific.md)** - Real-world compile patterns from 40+ biowasm tools (minimap2, samtools, bowtie2, fastp, bhtsne, MAFFT). Includes debugging checklist and quick reference table.
- **[references/dependencies.md](references/dependencies.md)** - Patterns for compiling common dependencies (htslib, zlib, boost, LZMA)
- **[references/skesa.md](references/skesa.md)** - SKESA-specific compilation notes and JavaScript integration examples

### External Resources

- Biowasm project: https://github.com/biowasm/biowasm
- Emscripten documentation: https://emscripten.org/docs/compiling/WebAssembly.html
