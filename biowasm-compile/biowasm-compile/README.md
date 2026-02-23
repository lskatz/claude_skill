# biowasm-compile

A Claude Code skill for compiling bioinformatics C/C++ tools to WebAssembly using Emscripten, following the [biowasm](https://github.com/biowasm/biowasm) pattern.

## What this skill does

Guides Claude in writing `compile.sh` scripts and integration code for compiling genomics tools (SKESA, Mash, minimap2, samtools, etc.) to WebAssembly for browser-based pipelines like [GenomicX](https://genomicx.github.io).

Includes real-world patterns from 40+ biowasm tools covering:
- SIMD optimization (minimap2)
- Autoconf-based tools (samtools, bcftools, htslib)
- Assembly instruction issues (bowtie2)
- Zlib linking fixes (fastp)
- ASYNCIFY for async operations (bhtsne)
- Memory configuration (MAFFT)
- Dependency compilation (htslib/LZMA)

## Installation

### For personal use (all projects)

Download the `.skill` file from the [releases](https://github.com/genomicx/claude_skill/releases) page and extract to your personal skills directory:

```bash
unzip biowasm-compile.skill -d ~/.claude/skills/
```

### For a specific project

Clone or download this skill to your project's skills directory:

```bash
git clone https://github.com/genomicx/claude_skill.git /tmp/claude_skill
cp -r /tmp/claude_skill/biowasm-compile/biowasm-compile ~/.claude/skills/
```

Or for project-specific installation:

```bash
mkdir -p .claude/skills
cp -r /tmp/claude_skill/biowasm-compile/biowasm-compile .claude/skills/
```

## Usage

Claude will automatically invoke this skill when you ask about compiling bioinformatics tools to WebAssembly, or you can invoke it directly:

```
/biowasm-compile minimap2
```

## Structure

```
biowasm-compile/
├── SKILL.md                     # Main guide — Emscripten flags, patterns, workflow
└── references/
    ├── tool-specific.md         # 40+ tool patterns with debugging guide (NEW)
    ├── dependencies.md          # Compiling common deps (htslib, zlib, boost)
    └── skesa.md                 # SKESA-specific compile notes and JS integration
```

## Context

Built for the [GenomicX](https://genomicx.github.io) ecosystem — browser-native bioinformatics tools with no installs, no uploads, no waiting.

## License

MIT
