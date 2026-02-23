# SKESA WebAssembly Compilation Notes

SKESA (Strategic K-mer Extension for Scrupulous Assemblies) by NCBI.
Source: https://github.com/ncbi/SKESA

## Key decisions

**Use `Makefile.nongs`** — this skips the NGS/SRA library dependency entirely.
SKESA can read from files without SRA support, which is all you need in a
browser context.

**Boost** — SKESA uses Boost for program_options, iterators, and hash
utilities. When compiling to WASM you replace `main()` with an exported
function, so `program_options` is irrelevant. The remaining Boost usage is
header-only. Use `-s USE_BOOST_HEADERS=1`.

**Threading** — SKESA is multi-threaded. Two options:
1. Single-thread build: patch `config.hpp` to set `MAX_THREADS=1` and remove
   OpenMP flags. Simpler, works everywhere.
2. WASM threads: add `-s USE_PTHREADS=1 -s PTHREAD_POOL_SIZE=4`. Requires
   COOP/COEP headers on the server. Vercel supports this.

## Compile script

```bash
#!/bin/bash
set -e

SKESA_VERSION="2.4.0"

# Download
wget -q https://github.com/ncbi/SKESA/archive/refs/tags/skesa.${SKESA_VERSION}.tar.gz
tar -xzf skesa.${SKESA_VERSION}.tar.gz
cd SKESA-skesa.${SKESA_VERSION}

# Patch: export assemble() instead of main()
# The JS wrapper will call Module.assemble(reads_r1, reads_r2) and get
# contigs back as a string via the virtual FS.
cat >> skesa.cpp << 'EOF'

#ifdef __EMSCRIPTEN__
#include <emscripten.h>
extern "C" {
EMSCRIPTEN_KEEPALIVE
int assemble(const char* r1_path, const char* r2_path, const char* out_path) {
    // Call SKESA assembly logic with file paths
    // Implementation wraps the core assembly function
    return skesa_assemble(r1_path, r2_path, out_path);
}
}
#endif
EOF

# Compile object files
em++ -O3 \
  -s USE_BOOST_HEADERS=1 \
  -s USE_ZLIB=1 \
  -DNDEBUG \
  -std=c++17 \
  -I. \
  -c skesa.cpp -o skesa.o

em++ -O3 \
  -s USE_BOOST_HEADERS=1 \
  -c kmercounter.cpp -o kmercounter.o

em++ -O3 \
  -s USE_BOOST_HEADERS=1 \
  -c glb_align.cpp -o glb_align.o

# Link
em++ skesa.o kmercounter.o glb_align.o \
  -O3 \
  -s WASM=1 \
  -s ALLOW_MEMORY_GROWTH=1 \
  -s MAXIMUM_MEMORY=4GB \
  -s MODULARIZE=1 \
  -s EXPORT_NAME='createSKESA' \
  -s EXPORTED_FUNCTIONS="['_assemble']" \
  -s EXPORTED_RUNTIME_METHODS="['FS','callMain']" \
  -s ENVIRONMENT='web,worker' \
  -s USE_BOOST_HEADERS=1 \
  -s USE_ZLIB=1 \
  -o /out/skesa.js
```

## JS integration

```javascript
import createSKESA from './skesa.js';

async function assembleReads(r1Buffer, r2Buffer) {
  const module = await createSKESA();

  // Write paired reads to virtual FS
  module.FS.writeFile('/r1.fastq', new Uint8Array(r1Buffer));
  module.FS.writeFile('/r2.fastq', new Uint8Array(r2Buffer));

  // Run assembly
  const result = module._assemble(
    module.allocateUTF8('/r1.fastq'),
    module.allocateUTF8('/r2.fastq'),
    module.allocateUTF8('/contigs.fasta')
  );

  // Read output
  if (result === 0) {
    return module.FS.readFile('/contigs.fasta', { encoding: 'utf8' });
  } else {
    throw new Error('SKESA assembly failed');
  }
}
```

## Expected output size

- `skesa.wasm`: ~4-8MB (before gzip)
- `skesa.js`: ~100KB glue code

## Known issues

- Large k-mer tables may hit the 4GB wasm32 memory ceiling for whole-genome
  assembly. For Consensusx (unmapped reads only) this is not a problem.
- The concurrent hash map (`concurrenthash.hpp`) uses atomics — these work in
  wasm32 but require `-matomics` flag if threading is enabled.
