# Common Dependency Compilation Patterns

## zlib
Built into emscripten ports — just use the flag, no manual compile needed:
```bash
em++ ... -s USE_ZLIB=1
```

## bzip2
```bash
em++ ... -s USE_BZIP2=1
```

## Boost (header-only)
Most bioinformatics Boost usage is header-only (program_options parsing is
not needed if you replace main() with an exported function):
```bash
em++ ... -s USE_BOOST_HEADERS=1
```

## htslib (for samtools, bcftools etc.)
```bash
git clone https://github.com/samtools/htslib
cd htslib
emconfigure ./configure --disable-bz2 --disable-lzma --disable-libcurl
emmake make -j$(nproc) lib-static
# Then link with: -L./htslib -lhts
```

## libdeflate
```bash
git clone https://github.com/ebiggers/libdeflate
cd libdeflate
emcmake cmake -B build -DLIBDEFLATE_BUILD_SHARED_LIB=OFF
emmake cmake --build build
```

## OpenSSL — avoid if possible
OpenSSL is painful under emscripten. If the tool only needs it for checksums,
patch it out or use emscripten's built-in crypto. If truly needed:
```bash
# Use a pre-built port
emcmake cmake ... -DOPENSSL_ROOT_DIR=/emsdk/upstream/emscripten/cache/sysroot
```
