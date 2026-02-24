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
Most bioinformatics Boost usage is header-only:
```bash
em++ ... -s USE_BOOST_HEADERS=1
```

## Boost (compiled libraries)
When tools need compiled Boost libraries (program_options, iostreams, system,
filesystem), build them with Emscripten's toolset. This was needed for SKESA:

```bash
# Download Boost
BOOST_VERSION="1_83_0"
wget -q "https://archives.boost.io/release/1.83.0/source/boost_${BOOST_VERSION}.tar.gz" -O boost.tar.gz
tar -xzf boost.tar.gz
cd boost_${BOOST_VERSION}

# Bootstrap and configure for Emscripten
./bootstrap.sh --with-libraries=program_options,iostreams,system
cat > project-config.jam << 'JAMEOF'
using gcc : emscripten : em++ ;
JAMEOF

# Build static libraries
./b2 toolset=gcc-emscripten link=static variant=release \
  cxxflags="-std=c++11 -O2 -s USE_ZLIB=1 -s USE_BZIP2=1" \
  --prefix=/tmp/boost_install install

# Then compile your tool with:
em++ ... -I/tmp/boost_install/include -L/tmp/boost_install/lib \
  -lboost_program_options -lboost_iostreams -lboost_system
```

Key points:
- Use `toolset=gcc-emscripten` with `em++` as the compiler
- Always use `link=static` (WASM can't do dynamic linking)
- Only build the libraries you actually need (`--with-libraries=...`)
- If the tool only uses Boost headers (no linking), prefer `-s USE_BOOST_HEADERS=1`

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
