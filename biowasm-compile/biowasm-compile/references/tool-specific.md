# Tool-Specific Compilation Patterns

Real-world compile patterns from the biowasm project. Use these as templates when compiling similar tools.

## Table of Contents

- [Simple Single-File Tools](#simple-single-file-tools)
- [Autoconf-Based Tools](#autoconf-based-tools)
- [SIMD Support](#simd-support)
- [Assembly Instruction Issues](#assembly-instruction-issues)
- [Zlib Linking Issues](#zlib-linking-issues)
- [ASYNCIFY for Async Operations](#asyncify-for-async-operations)
- [Memory Configuration](#memory-configuration)
- [Dependency Compilation](#dependency-compilation)
- [Data Preprocessing](#data-preprocessing)

---

## Simple Single-File Tools

### seqtk Pattern

For simple C tools with no dependencies:

```bash
#!/bin/bash
emcc seqtk.c \
    -o ../build/seqtk.js \
    -O2 \
    $EM_FLAGS
```

**When to use:**
- Single C file
- No external dependencies
- No complex build system

**Tools using this pattern:**
- seqtk

---

## Autoconf-Based Tools

### samtools/bcftools Pattern

Tools using autoconf require configuration before compilation:

```bash
#!/bin/bash

# Generate configure script
autoheader
autoconf -Wno-syntax  # -Wno-syntax suppresses autoconf warnings

# Configure with emscripten
emconfigure ./configure \
    --without-curses \
    --with-htslib="../../htslib/src/" \
    CFLAGS="-s USE_ZLIB=1 -s USE_BZIP2=1"

# Build
emmake make samtools CC=emcc AR=emar \
    CFLAGS="-O2 -s USE_ZLIB=1 -s USE_BZIP2=1 $CFLAGS_LZMA" \
    LDFLAGS="$EM_FLAGS --preload-file examples/@/samtools/examples/ \
             -s ERROR_ON_UNDEFINED_SYMBOLS=0 -O2 $LDFLAGS_LZMA"
```

**Key flags:**
- `-s ERROR_ON_UNDEFINED_SYMBOLS=0` - Required! htslib tools have undefined symbols that resolve at link time
- `--without-curses` - Disable terminal UI dependencies
- `--with-htslib=path` - Link against custom htslib build

**Tools using this pattern:**
- samtools
- bcftools
- htslib (tabix, bgzip, htsfile)

**Common issues:**
- Missing autoconf: Install with `apt-get install autoconf`
- Curses linking errors: Use `--without-curses`
- Undefined symbol errors: Add `-s ERROR_ON_UNDEFINED_SYMBOLS=0`

---

## SIMD Support

### minimap2 Pattern

Some tools have SIMD-optimized code paths. Compile both versions for browser compatibility:

```bash
#!/bin/bash

WASM_FLAGS="$EM_FLAGS --preload-file test/@/minimap2/"

# Non-SIMD version (maximum compatibility)
echo "Compiling without SIMD"
make clean
emmake make \
    -f Makefile.simde \
    PROGRAM="minimap2" \
    CFLAGS="-O2 -Wno-pass-failed -Wno-return-type" \
    WASM_FLAGS="$WASM_FLAGS" \
    minimap2

# SIMD version (faster on modern browsers)
echo "Compiling with SIMD"
make clean
emmake make \
    -f Makefile \
    PROGRAM="minimap2-simd" \
    CFLAGS="-O2 -Wno-return-type -Wno-unused-command-line-argument -msimd128" \
    WASM_FLAGS="$WASM_FLAGS" \
    minimap2-simd
```

**SIMD flag:** `-msimd128`

**Why both versions:**
- SIMD version: 2-3x faster on Chrome/Firefox/Safari with SIMD support
- Non-SIMD fallback: Works on older browsers or browsers with SIMD disabled

**Tools with SIMD:**
- minimap2 (uses Makefile.simde for non-SIMD)
- Any tool using SSE/AVX intrinsics

**Common warnings to suppress:**
- `-Wno-pass-failed` - SIMD optimization passes may fail
- `-Wno-return-type` - Macro expansions sometimes generate false positives

---

## Assembly Instruction Issues

### bowtie2 Pattern

Tools using x86-specific assembly instructions need special handling:

```bash
#!/bin/bash

# Build:
#   - NO_TBB=1: disable Threading Building Blocks library
#   - POPCNT_CAPABILITY=0: popcnt is an x86 assembly instruction; not in WebAssembly
emmake make bowtie2-align-s \
    NO_TBB=1 \
    POPCNT_CAPABILITY=0 \
    WASM_FLAGS="$EM_FLAGS --preload-file example/@/bowtie2/example/ \
               -s ASSERTIONS=1 -s DISABLE_EXCEPTION_CATCHING=0"
```

**Common x86 instructions to disable:**
- `POPCNT` - Population count (count set bits)
- `SSE`, `SSE2`, `SSE4.1` - SIMD extensions (use `-msimd128` instead)
- `AVX`, `AVX2` - Advanced vector extensions

**How to find them:**
- Look for `#ifdef __x86_64__` or `#ifdef __SSE__` in source
- Search Makefile for capability flags
- Check compiler errors for unknown instructions

**Tools needing this:**
- bowtie2 (POPCNT)
- Any tool with x86-specific optimizations

---

## Zlib Linking Issues

### fastp Pattern

Function signature mismatches between bundled zlib and Emscripten's zlib:

```bash
#!/bin/bash

# Set `DYNAMIC_ZLIB` to use Emscripten's zlib instead of bundled version
# This prevents the warning:
#   wasm-ld: warning: function signature mismatch: gzoffset
#   >>> defined as (i32) -> i32 in ./obj/fastqreader.o
#   >>> defined as (i32) -> i64 in libz.a(gzlib.c.o)
# Without this fix, fastp crashes on .fastq.gz files.

emmake make \
    TARGET="../build/fastp.js" \
    CXXFLAGS="-std=c++11 -O3 -DDYNAMIC_ZLIB -s USE_ZLIB=1" \
    LIBS="-s USE_ZLIB=1 $EM_FLAGS --preload-file testdata@/fastp/testdata"
```

**The problem:**
- Tool bundles its own zlib
- Emscripten provides zlib with different function signatures
- Linker sees conflicting definitions
- Tool crashes at runtime with `.gz` files

**The fix:**
- Define `-DDYNAMIC_ZLIB` or equivalent flag
- Add `-s USE_ZLIB=1` to both compile and link flags
- Check tool's build system for zlib config options

**Tools affected:**
- fastp
- Any tool bundling zlib, bzip2, or other common libs

**How to diagnose:**
- Look for `function signature mismatch` warnings
- Check if tool crashes on compressed input
- Search for bundled `zlib/` directory in source

---

## ASYNCIFY for Async Operations

### bhtsne Pattern

Tools that need to call async JavaScript functions from C/C++:

```bash
#!/bin/bash

FLAGS=$(cat <<EOF
    $EM_FLAGS \
    -O3 \
    -s ASYNCIFY=1 \
    -s 'ASYNCIFY_IMPORTS=["send_names","send_results"]' \
    -s EXPORTED_RUNTIME_METHODS=["getValue","UTF8ToString","callMain","FS","PROXYFS","WORKERFS"] \
    --preload-file ../data/brain8.snd@/bhtsne/brain8.snd
EOF
)

emmake make \
    CC=emcc CXX=em++ \
    CFLAGS="-O2 -s USE_ZLIB=1 -w" \
    LIBS="-s USE_ZLIB=1 -lm $FLAGS"
```

**ASYNCIFY flags:**
- `-s ASYNCIFY=1` - Enable async transformation
- `-s ASYNCIFY_IMPORTS=["func1","func2"]` - List of async JS functions called from C

**When to use:**
- Tool needs to call async JavaScript APIs (fetch, setTimeout)
- Progress callbacks to update UI
- Streaming I/O operations
- Web Worker communication

**Cost:**
- Increases binary size (~30-50KB)
- Slightly slower runtime (stack unwinding overhead)
- Use only when necessary!

**Tools using ASYNCIFY:**
- bhtsne (progress reporting)

**Alternative:** Use Emscripten's `-s PROXY_TO_PTHREAD=1` for blocking operations in workers

---

## Memory Configuration

### MAFFT Pattern

Tools needing more than default 16MB memory:

```bash
cd core
emmake make CC="emcc \
    -s TOTAL_MEMORY=360MB \
    -s ASSERTIONS=1 \
    -s USE_PTHREADS=0 \
    $EM_FLAGS"
```

**Memory flags (old style - pre-3.1.29):**
- `-s TOTAL_MEMORY=360MB` - Fixed memory (must be multiple of 64KB)
- Use with tools that allocate large arrays upfront

**Memory flags (modern - 3.1.29+):**
- `-s INITIAL_MEMORY=64MB` - Starting heap
- `-s ALLOW_MEMORY_GROWTH=1` - Grow automatically (recommended)
- `-s MAXIMUM_MEMORY=4GB` - Max for wasm32, 16GB for MEMORY64

**Which to use:**
- Most genomics tools: Use `ALLOW_MEMORY_GROWTH=1` + `MAXIMUM_MEMORY=4GB`
- Tools with known memory needs: Set `INITIAL_MEMORY` to avoid growth overhead
- Never use `TOTAL_MEMORY` with `ALLOW_MEMORY_GROWTH`!

**Tools with explicit memory:**
- MAFFT (360MB fixed)

---

## Dependency Compilation

### htslib/LZMA Pattern

Compiling dependencies from source before the main tool:

```bash
#!/bin/bash

# Compile LZMA dependency
LZMA_VERSION="5.2.5"
curl -LO "https://tukaani.org/xz/xz-${LZMA_VERSION}.tar.gz"
tar -xf xz-${LZMA_VERSION}.tar.gz
cd xz-${LZMA_VERSION}
emconfigure ./configure --disable-shared --disable-threads
emmake make -j4 CFLAGS="-Oz -fPIC -s USE_PTHREADS=0 -s EXPORT_ALL=1 -s ASSERTIONS=1"
cd -

# Now build main tool with LZMA
CFLAGS="-s USE_ZLIB=1 -s USE_BZIP2=1 ${CFLAGS_LZMA}"
LDFLAGS="$LDFLAGS_LZMA"
emconfigure ./configure CFLAGS="$CFLAGS" LDFLAGS="$LDFLAGS"
emmake make CC=emcc AR=emar CFLAGS="-O2 $CFLAGS" LDFLAGS="$EM_FLAGS -O2 $LDFLAGS"
```

**Key patterns:**
- `--disable-shared` - Static linking (WASM can't do dynamic linking)
- `--disable-threads` - Avoid pthread complexity unless needed
- `-Oz` - Optimize for size (important for WASM)
- `-fPIC` - Position independent code
- `-s EXPORT_ALL=1` - Make all functions available to linker

**Common dependencies:**
- LZMA/XZ - See above pattern
- Zlib - Use `-s USE_ZLIB=1` (built into Emscripten)
- Bzip2 - Use `-s USE_BZIP2=1` (built into Emscripten)
- Boost - Use `-s USE_BOOST_HEADERS=1` for header-only libs

**See also:** `references/dependencies.md` for htslib, zlib, boost patterns

---

## Data Preprocessing

### bowtie2 Pattern

Reduce preloaded data size for faster loading:

```bash
#!/bin/bash

# Remove large unneeded files
rm example/reads/combined_reads.bam
rm example/reads/longreads.fq

# Truncate test files to first 100 lines
head -n100 example/reads/reads_1.fq > example/reads/reads_1.fq.tmp
head -n100 example/reads/reads_2.fq > example/reads/reads_2.fq.tmp
mv example/reads/reads_1.fq.tmp example/reads/reads_1.fq
mv example/reads/reads_2.fq.tmp example/reads/reads_2.fq

# Now compile with smaller preload-file
emmake make bowtie2-align-s \
    WASM_FLAGS="$EM_FLAGS --preload-file example/@/bowtie2/example/"
```

**Why this matters:**
- `.data` files load synchronously on page load
- Large data files = slow initial load
- Users can upload their own files at runtime

**Best practices:**
- Keep preloaded data <1MB if possible
- Use minimal test datasets
- Compress reference files (`.gz`)
- Document that users can upload full datasets

**Tools doing this:**
- bowtie2 (truncates test reads)
- bhtsne (gzips after compile)

---

## Quick Reference Table

| Tool | Pattern | Key Flags | Notes |
|------|---------|-----------|-------|
| seqtk | Simple C | `-O2` | Single file, no deps |
| samtools | Autoconf | `ERROR_ON_UNDEFINED_SYMBOLS=0` | Needs htslib |
| bcftools | Autoconf | `ERROR_ON_UNDEFINED_SYMBOLS=0` | Needs htslib |
| minimap2 | SIMD | `-msimd128`, `Makefile.simde` | Compile both versions |
| bowtie2 | Assembly | `POPCNT_CAPABILITY=0`, `NO_TBB=1` | Disable x86 instructions |
| fastp | Zlib fix | `-DDYNAMIC_ZLIB`, `-s USE_ZLIB=1` | Avoid zlib signature mismatch |
| bhtsne | Async | `-s ASYNCIFY=1` | For async JS calls |
| MAFFT | Memory | `-s TOTAL_MEMORY=360MB` | Fixed memory allocation |
| htslib | Dependency | See pattern | Compiles LZMA from source |

---

## Debugging Checklist

When a compile fails, check:

1. **Missing autoconf?** → Install: `apt-get install autoconf`
2. **Undefined symbols?** → Add: `-s ERROR_ON_UNDEFINED_SYMBOLS=0`
3. **Assembly errors?** → Look for x86 capability flags to disable
4. **Zlib crashes?** → Use `-DDYNAMIC_ZLIB -s USE_ZLIB=1`
5. **Out of memory?** → Increase `MAXIMUM_MEMORY` or use `ALLOW_MEMORY_GROWTH=1`
6. **Slow page load?** → Reduce preloaded data size
7. **Threading errors?** → Add `NO_TBB=1`, `USE_PTHREADS=0`, or `--disable-threads`
8. **SIMD compile errors?** → Try non-SIMD Makefile first

---

## See Also

- [dependencies.md](dependencies.md) - Common dependency compile patterns
- [skesa.md](skesa.md) - SKESA-specific notes
- Main SKILL.md - General Emscripten flags and workflow
