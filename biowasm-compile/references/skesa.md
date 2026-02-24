# SKESA WebAssembly Compilation Notes

SKESA (Strategic K-mer Extension for Scrupulous Assemblies) by NCBI.
Source: https://github.com/ncbi/SKESA
Working WASM build: https://github.com/genomicx/skesa-wasm

## Key decisions

**Use `Makefile.nongs`** — this skips the NGS/SRA library dependency entirely.
SKESA can read from files without SRA support, which is all you need in a
browser context.

**Boost** — SKESA needs compiled Boost libraries (program_options, iostreams,
system), not just headers. Build them with Emscripten using the
`gcc-emscripten` toolset (see compile script below).

**Threading** — SKESA is multi-threaded. For WASM, force single-threaded by
adding `#ifdef __EMSCRIPTEN__` guard in `skesa.cpp` to set `ncores=1`.
Don't use `-s USE_PTHREADS=1` — it adds complexity and requires COOP/COEP
headers.

**C++ standard** — SKESA uses `std=c++11`. Use `-Du_int64_t=uint64_t` and
`-Du_int32_t=uint32_t` since Emscripten doesn't define the BSD typedefs.

**Exceptions** — SKESA uses C++ exceptions. Both `-fexceptions` in CXXFLAGS
and `-s DISABLE_EXCEPTION_CATCHING=0` in LDFLAGS are required.

## Critical wasm32 fixes

SKESA compiles successfully but produces **completely wrong results** without
these source code patches. These are the most important part of the port.

### Fix 1: Read storage bit corruption (common_util.hpp)

In `CReadHolder::PushBack()`, nucleotides are packed into `uint64_t` words
using left shifts up to 62. The shift operand comes from
`find(bin2NT.begin(), bin2NT.end(), *it) - bin2NT.begin()` which returns
`ptrdiff_t` — 32-bit on wasm32. Left-shifting a 32-bit value by >= 32 bits
is undefined behavior, and WASM masks the shift to 5 bits.

**Result:** Half of all nucleotides in read storage are silently corrupted.
This causes ~31% fewer distinct kmers and complete assembly failure.

```cpp
// BUG (two overloads of PushBack):
m_storage.back() += ((find(bin2NT.begin(), bin2NT.end(), *it) - bin2NT.begin()) << shift);

// FIX: cast to uint64_t before shifting
m_storage.back() += (uint64_t(find(bin2NT.begin(), bin2NT.end(), *it) - bin2NT.begin()) << shift);
```

### Fix 2: Kmer count packing overflow (KmerInit.hpp, counter.hpp)

SKESA packs strand info (16 bits), branch info (8 bits), and kmer count
(32 bits) into a single 64-bit value using bit shifts:

```cpp
Kmers().UpdateCount((plusf << 48) + (b << 32) + total_count, index);
```

The pair type in `KmerInit.hpp` uses `size_t` for the count:

```cpp
template<int N> using TLargeIntVec = vector<pair<LargeInt<N>, size_t>>;
```

On wasm32, `size_t` is 32-bit, so the `<< 48` and `<< 32` shifts overflow.

**Fix:** Change `size_t` to `uint64_t` in:
- `KmerInit.hpp`: pair type definition
- `counter.hpp`: All count-related interfaces (PushBack, UpdateCount,
  GetCount, GetKmerCount, and their visitor structs: push_back, update_count,
  get_count, get_kmer_count)
- `counter.hpp` GetBranches(): local variables for branch/count packing
- `counter.hpp` SpawnKmersJob(): strand counting (`count += uint64_t(1) << 32`)

### Fix 3: Force single-threaded (skesa.cpp)

```cpp
#ifdef __EMSCRIPTEN__
    ncores = 1;
#endif
```

Add after the `ncores` variable is set from command-line arguments.

## Docker-based compile script

The compilation uses a Docker container for reproducibility. Two files:

### compile-docker.sh (host script)

```bash
#!/bin/bash
set -e
docker run --rm --platform linux/amd64 \
  -v "$(pwd):/work" \
  emscripten/emsdk:3.1.50 \
  bash /work/compile.sh
```

### compile.sh (runs inside Docker)

```bash
#!/bin/bash
set -e

SKESA_VERSION="2.5.1"
BOOST_VERSION="1_83_0"
BOOST_URL="https://archives.boost.io/release/1.83.0/source/boost_${BOOST_VERSION}.tar.gz"

SRC=/work/src
BUILD=/work/build
mkdir -p $BUILD

# =====================================================
# 1. Build Boost compiled libraries for Emscripten
# =====================================================
cd /tmp
wget -q "$BOOST_URL" -O boost.tar.gz
tar -xzf boost.tar.gz
cd boost_${BOOST_VERSION}

# Bootstrap and configure for Emscripten
./bootstrap.sh --with-libraries=program_options,iostreams,system
cat > project-config.jam << 'JAMEOF'
using gcc : emscripten : em++ ;
JAMEOF

./b2 toolset=gcc-emscripten link=static variant=release \
  cxxflags="-std=c++11 -O2 -s USE_ZLIB=1 -s USE_BZIP2=1" \
  --prefix=/tmp/boost_install install 2>&1 | tail -5

BOOST_INC=/tmp/boost_install/include
BOOST_LIB=/tmp/boost_install/lib

# =====================================================
# 2. Compile SKESA source files
# =====================================================
cd $SRC

CXXFLAGS="-std=c++11 -O2 -D NO_NGS -fexceptions \
  -Du_int64_t=uint64_t -Du_int32_t=uint32_t \
  -I${BOOST_INC} -s USE_ZLIB=1 -s USE_BZIP2=1"

echo "[1/5] Compiling skesa.cpp..."
em++ $CXXFLAGS -c skesa.cpp -o skesa.o

echo "[2/5] Compiling kmercounter.cpp..."
em++ $CXXFLAGS -c kmercounter.cpp -o kmercounter.o

echo "[3/5] Compiling glb_align.cpp..."
em++ $CXXFLAGS -c glb_align.cpp -o glb_align.o

# =====================================================
# 3. Link
# =====================================================
echo "[4/5] Linking SKESA..."
em++ skesa.o kmercounter.o glb_align.o \
  -O2 -fexceptions \
  -L${BOOST_LIB} \
  -lboost_program_options -lboost_iostreams -lboost_system \
  -s USE_ZLIB=1 -s USE_BZIP2=1 \
  -s WASM=1 \
  -s MODULARIZE=1 \
  -s EXPORT_NAME="createSKESA" \
  -s ALLOW_MEMORY_GROWTH=1 \
  -s MAXIMUM_MEMORY=4GB \
  -s INITIAL_MEMORY=268435456 \
  -s EXPORTED_FUNCTIONS="['_main']" \
  -s EXPORTED_RUNTIME_METHODS="['callMain','FS']" \
  -s ENVIRONMENT='web,worker,node' \
  -s DISABLE_EXCEPTION_CATCHING=0 \
  -o ${BUILD}/skesa.js

echo "[5/5] Build complete!"
ls -lh ${BUILD}/skesa.js ${BUILD}/skesa.wasm
```

## JS integration (Node.js)

```javascript
import { readFileSync } from 'fs';
import { gunzipSync } from 'zlib';

const { default: createSKESA } = await import('./build/skesa.js');

const stdout = [];
const stderr = [];
const skesa = await createSKESA({
  wasmBinary: readFileSync('./build/skesa.wasm'),
  noInitialRun: true,
  print: (t) => stdout.push(t),
  printErr: (t) => stderr.push(t),
});

// Write reads to virtual filesystem
const r1 = gunzipSync(readFileSync('reads_R1.fastq.gz'));
const r2 = gunzipSync(readFileSync('reads_R2.fastq.gz'));
skesa.FS.writeFile('/r1.fastq', r1);
skesa.FS.writeFile('/r2.fastq', r2);

// Run assembly
try {
  skesa.callMain([
    '--reads', '/r1.fastq,/r2.fastq',
    '--cores', '1',
    '--min_contig', '200',
  ]);
} catch (err) { /* SKESA may throw on exit */ }

// Parse FASTA output from stdout
const fasta = stdout.join('\n');
```

## JS integration (Browser)

```javascript
const skesa = await createSKESA({
  noInitialRun: true,
  print: (t) => stdout.push(t),
  printErr: (t) => stderr.push(t),
});

// Decompress gzipped reads in browser
const ds = new DecompressionStream('gzip');
const blob = new Blob([r1Buffer]);
const r1Data = new Uint8Array(await new Response(blob.stream().pipeThrough(ds)).arrayBuffer());

skesa.FS.writeFile('/r1.fastq', r1Data);
skesa.FS.writeFile('/r2.fastq', r2Data);

try {
  skesa.callMain(['--reads', '/r1.fastq,/r2.fastq', '--cores', '1', '--min_contig', '200']);
} catch (err) {}
```

## Output sizes

- `skesa.wasm`: ~2.2MB
- `skesa.js`: ~86KB glue code

## Verified results

Assembly results matching native x86-64 SKESA exactly:

| Dataset | Contigs | Total bp | N50 |
|---------|---------|----------|-----|
| T7 phage (~40kb genome) | 1 | 39,751bp | 39,751bp |
| T4 phage (~169kb genome) | 1 | 165,823bp | 165,823bp |
| Bacteria (500K reads, ~5Mb genome) | 156 | 5.3Mbp | 74,949bp |

Kmer counts match native exactly: 5,285,820 distinct from 200K paired reads.

## Debugging approach used

When SKESA compiled but produced 0 contigs:

1. Added `#ifdef __EMSCRIPTEN__` debug output to histogram analysis
   (`CalculateGenomeSize`) — found corrupted kmer frequency histogram
2. Compared native vs WASM kmer counts — found 31% fewer distinct kmers
3. Added debug output to `SpawnKmersJob` and `extract_uniq` — confirmed raw
   kmer generation was correct but dedup produced different results
4. Traced the data flow: read → bit-packed storage → kmer extraction →
   sort → dedup. Found the `ptrdiff_t << shift` UB in PushBack
5. After fixing PushBack, kmer counts matched native exactly

Key insight: **compile success ≠ correct results**. Always validate output
against native builds, especially for tools with bit-packing or `size_t` usage.
