# biowasm-compile skill

A Claude skill for compiling bioinformatics C/C++ tools to WebAssembly using Emscripten, following the [biowasm](https://github.com/biowasm/biowasm) pattern.

## What this skill does

Helps Claude write `compile.sh` scripts and integration code for compiling genomics tools (SKESA, Mash, minimap2, samtools, etc.) to WebAssembly for browser-based pipelines like [GenomicX](https://genomicx.github.io).

## Structure

```
biowasm-compile/
├── SKILL.md                     # Main skill — Emscripten flags, patterns, debugging
└── references/
    ├── dependencies.md          # Compiling common deps (htslib, zlib, boost)
    └── skesa.md                 # SKESA-specific compile notes and JS integration
```

## Installation

Add to your Claude skills directory or upload as a `.skill` file.

## Context

Built as part of the [GenomicX](https://genomicx.github.io) ecosystem — browser-native bioinformatics tools with no installs, no uploads, no waiting.
