# Impressx — EMBOSS Sequence Analysis Tools in the Browser

## Overview

Impressx brings classic EBI-EMBOSS sequence analysis utilities to the browser.
Run widely-used tools for pairwise alignment, ORF finding, restriction
analysis, sequence translation, and motif searching — all client-side with no
data leaving your machine.

All computation runs client-side via WebAssembly or pure JavaScript.

## Problem

EMBOSS (European Molecular Biology Open Software Suite) has been a workhorse
of sequence analysis for over two decades. Its ~200 tools cover everything
from pairwise alignment to restriction mapping. However:

- EMBOSS is a C codebase last substantially updated around 2013–2017. It
  still works but installation is increasingly painful on modern systems
  (build issues on macOS, dependency conflicts).
- The EBI EMBOSS web services have been deprecated or are poorly maintained.
- Students and clinicians who need a quick sequence operation (translate, find
  ORFs, reverse complement, align two sequences) have to either install the
  full suite or find scattered web tools of variable quality.

The individual algorithms are well-defined and relatively simple. Rather than
compiling all of EMBOSS to WebAssembly, Impressx implements the most useful
operations as a clean, modern web interface — either via targeted WASM
compilation of specific EMBOSS programs, or via faithful JavaScript
reimplementations of the underlying algorithms.

## Core Concept

```
Sequence(s) (FASTA / raw text)
    |
    v  [select tool from sidebar]
    |
    v  [run analysis — JS or WASM]
    |
    v
Formatted output (alignment, ORF list, restriction map, etc.)
```

Impressx is a **toolkit** — a collection of small, focused utilities under
one roof, rather than a single pipeline. Think of it as a Swiss Army knife for
sequence manipulation.

## Target Users

- **Students** doing coursework assignments (align two sequences, find ORFs
  in a gene, translate a CDS)
- **Bench scientists** who need quick sequence operations without opening a
  terminal
- **Bioinformaticians** who want a fast utility without installing EMBOSS
- **Anyone** who currently Googles "reverse complement online" or "restriction
  sites online"

## Tool Selection

From EMBOSS's ~200 tools, prioritize the ones that are most commonly used and
most useful for a genomics audience. Grouped by category:

### Tier 1: MVP (highest value, simplest to implement)

| Tool       | EMBOSS equivalent | What it does                       |
| ---------- | ----------------- | ---------------------------------- |
| **Translate** | `transeq`      | Translate DNA to protein (6-frame) |
| **Reverse Complement** | `revseq` | Reverse complement a DNA sequence |
| **ORF Finder** | `getorf`     | Find all open reading frames       |
| **Restriction Map** | `restrict` | Find restriction enzyme cut sites |
| **Sequence Stats** | `infoseq`/`compseq` | Length, GC%, base composition |
| **Needle** | `needle`          | Global pairwise alignment (Needleman-Wunsch) |
| **Water**  | `water`           | Local pairwise alignment (Smith-Waterman) |

### Tier 2: Post-MVP

| Tool       | EMBOSS equivalent | What it does                       |
| ---------- | ----------------- | ---------------------------------- |
| **Dotplot** | `dotmatcher`     | Dot matrix comparison of two sequences |
| **Primer Search** | `primersearch` | Find primer binding sites        |
| **Motif Search** | `fuzzpro`/`fuzznuc` | Search for sequence patterns / PROSITE motifs |
| **CpG Islands** | `cpgplot`    | Identify CpG islands               |
| **Codon Usage** | `cusp`/`codcmp` | Codon usage table + comparison  |
| **Sequence Extract** | `extractseq` | Extract subsequence by coordinates |
| **Format Convert** | `seqret`  | Convert between FASTA, GenBank, EMBL |

### Tier 3: Future

| Tool       | EMBOSS equivalent | What it does                       |
| ---------- | ----------------- | ---------------------------------- |
| **Multiple Alignment Viewer** | — | Render MSA from FASTA/Clustal input |
| **Protein Properties** | `pepstats` | MW, pI, amino acid composition |
| **Hydropathy Plot** | `pepwindow` | Kyte-Doolittle hydrophobicity |
| **Signal Peptide** | —          | Predict signal peptides (simple heuristic) |

## Implementation Strategy

### Pure JavaScript (preferred for most tools)

The underlying algorithms are well-documented and straightforward to implement
in TypeScript:

- **Needleman-Wunsch** (global alignment): O(mn) dynamic programming. Standard
  textbook algorithm. Scoring matrices (BLOSUM62, EDNAFULL) shipped as JSON
  lookup tables.
- **Smith-Waterman** (local alignment): Same DP approach, traceback differs.
- **6-frame translation**: Codon lookup table, iterate over reading frames.
  Trivial.
- **Reverse complement**: Character mapping. Trivial.
- **ORF finding**: Scan for start codons (ATG), extend to stop codons
  (TAA/TAG/TGA), filter by minimum length. Straightforward.
- **Restriction sites**: Pre-built database of enzyme recognition sequences
  (REBASE). Regex or exact string matching along the sequence.
- **Base composition / GC%**: Single-pass character counting.

For sequences up to ~10 Mb (a full bacterial genome), all of these run in
< 1 second in JavaScript. No WASM needed.

### WebAssembly (for performance-critical cases)

If alignment of very large sequences becomes a bottleneck:

- Use **parasail** (SIMD-accelerated alignment library, C) compiled to WASM
  for Needleman-Wunsch / Smith-Waterman on sequences > 100 kb
- Or use **MAFFT** (already in Biowasm) for multiple alignment
- EMBOSS itself is C code and could be selectively compiled, but the build
  system is complex and it's easier to reimplement individual algorithms

### Data Files

- **REBASE** restriction enzyme database: ~300 common enzymes with recognition
  sequences and cut positions. Ship as JSON (~50 KB).
- **Scoring matrices**: BLOSUM45, BLOSUM62, BLOSUM80, PAM250, EDNAFULL. Ship
  as JSON (~20 KB each).
- **Codon tables**: NCBI genetic code tables (standard, bacterial, mitochondrial,
  etc.). Ship as JSON (~5 KB).

Total data footprint: < 500 KB.

## User Interface

### Layout

Impressx uses a **sidebar + workspace** layout, different from the other
GenomicX tools:

```
┌──────────┬────────────────────────────────────────┐
│          │                                        │
│  Tool    │  Input                                 │
│  list    │  [sequence input area]                 │
│          │                                        │
│  -------─│  Settings                              │
│  Sequence│  [tool-specific parameters]            │
│  -------─│                                        │
│  Alignment│  [Run]                                │
│  -------─│                                        │
│  Analysis│  Output                                │
│  -------─│  [formatted results]                   │
│          │                                        │
└──────────┴────────────────────────────────────────┘
```

### Sequence Input
- Text area for pasting sequence(s) in FASTA or raw format
- File upload for larger sequences
- Detect DNA vs. protein automatically from character content
- Persist last-used sequence in sessionStorage for convenience

### Tool-Specific Panels

**Translate:**
- Frame selection: all 6 frames / forward 3 / reverse 3 / specific frame
- Genetic code: Standard (default), Bacterial, Mitochondrial, etc.
- Output: protein sequences with frame labels, ORF highlighting

**Needle / Water (Pairwise Alignment):**
- Two sequence input areas (or split from multi-FASTA)
- Matrix selection: BLOSUM62 (protein), EDNAFULL (DNA)
- Gap open / extend penalties (defaults: 10 / 0.5)
- Output: formatted alignment with identity/similarity scores, alignment
  length, gaps

**ORF Finder:**
- Minimum ORF length: 100 bp (default, configurable)
- Frames to search: all 6 / forward only
- Output: table of ORFs (frame, start, end, length, protein sequence),
  graphical map showing ORF positions

**Restriction Map:**
- Enzyme selection: common enzymes / all enzymes / custom list
- Minimum cut sites: 1 (show all) or user-configured
- Output: linear map with cut site positions marked, table of enzymes with
  fragment sizes, number of cuts

### Output
- Formatted text (monospace, coloured alignment)
- Copy-to-clipboard button
- Download as plain text / FASTA / CSV where appropriate
- Results stay visible while switching tools (tabbed output)

## Tech Stack

| Component        | Tool / Library                         |
| ---------------- | -------------------------------------- |
| Alignment (JS)   | Custom Needleman-Wunsch / Smith-Waterman |
| Alignment (WASM) | parasail (optional, for large seqs)    |
| MSA (optional)   | MAFFT (Biowasm, Tier 3)               |
| Restriction DB   | REBASE (JSON)                          |
| Scoring matrices | BLOSUM62, EDNAFULL, etc. (JSON)        |
| Frontend         | TypeScript, Vite, React                |
| Styling          | GenomicX design system (ronaQC tokens) |

## Performance Considerations

Most tools are near-instant for typical inputs:

| Operation         | Typical input         | Expected time |
| ----------------- | --------------------- | ------------- |
| Translate         | 5 kb gene             | < 10 ms       |
| Reverse complement| 5 Mb genome           | < 100 ms      |
| ORF finder        | 5 Mb genome           | < 500 ms      |
| Restriction map   | 5 Mb genome, 300 enzymes | < 2 s      |
| Needle (global)   | 2 x 1 kb sequences   | < 100 ms      |
| Needle (global)   | 2 x 10 kb sequences  | < 5 s (JS) / < 1 s (WASM) |
| Water (local)     | 2 x 1 kb sequences   | < 100 ms      |
| Base composition  | 5 Mb genome           | < 100 ms      |

Memory: all operations fit in < 100 MB for bacterial-scale sequences.

The only bottleneck is pairwise alignment of very long sequences (> 10 kb),
where the O(mn) DP matrix becomes large. For these cases, offer a banded
alignment option (reducing to near-linear time) or use a WASM-compiled
aligner.

## Relationship to Other GenomicX Tools

Impressx is a **utility toolkit** — it complements the other tools rather
than feeding into a pipeline:

- A researcher using **Genetrax** finds an AMR gene → uses Impressx to
  **translate** it and check the protein sequence
- A student using **MLSTx** wants to **align** two allele sequences to see
  the SNP differences
- Someone using **BRIGx** wants to **find restriction sites** in a region of
  interest
- A user wants to **extract and reverse complement** a subsequence before
  uploading to another tool

Impressx is the "lab bench" companion to the specialized analysis tools.

## MVP Scope (v1)

Implement Tier 1 tools only:

1. Translate (6-frame translation)
2. Reverse complement
3. ORF finder
4. Restriction map (top 50 common enzymes)
5. Sequence stats (length, GC%, base composition)
6. Needle (global pairwise alignment)
7. Water (local pairwise alignment)

All implemented in pure JavaScript. No WASM dependency in MVP.

**Not in MVP:**
- Dotplot, primer search, motif search
- WASM-accelerated alignment
- Multiple alignment
- Protein analysis tools
- Format conversion

## Prior Art

| Tool              | Scope              | Runs in browser? |
| ----------------- | ------------------ | ---------------- |
| EMBOSS suite      | ~200 seq tools     | No (C, CLI)      |
| EBI EMBOSS web    | EMBOSS wrappers    | Server-side (deprecated) |
| Galaxy EMBOSS     | EMBOSS wrappers    | Server-side      |
| ExPASy Translate  | Translation only   | Server-side      |
| NEBcutter         | Restriction only   | Server-side      |
| Reverse Complement (various) | RevComp only | Mixed |
| **Impressx**      | **Core EMBOSS set**| **Yes (client-side JS)** |

No existing tool provides a unified, client-side collection of the most-used
EMBOSS-equivalent utilities.

## Suggested apps.json Entry

```json
{
  "id": "impressx",
  "name": "Impressx",
  "tagline": "EMBOSS Tools in the Browser",
  "description": "Run classic EBI-EMBOSS sequence analysis utilities directly in your browser. Access widely-used tools for sequence alignment, restriction analysis, and motif searching — all client-side with no uploads to external servers.",
  "icon": "wrench",
  "tech": ["JavaScript", "TypeScript", "Vite", "React"],
  "features": [
    "Pairwise alignment (Needleman-Wunsch, Smith-Waterman)",
    "6-frame translation and ORF finding",
    "Restriction enzyme mapping",
    "Fully client-side, no data leaves your machine"
  ],
  "demoUrl": "",
  "sourceUrl": "",
  "color": "#14B8A6",
  "status": {
    "scoping": "in-progress",
    "pipeline": "not-started",
    "webDev": "not-started",
    "benchmarking": "not-started"
  }
}
```
