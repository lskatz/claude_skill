# pMLSTx — Browser-Based Plasmid MLST Typing

## Overview

pMLSTx types plasmid replicons using pMLST schemes directly in the browser.
Upload bacterial genome assemblies to identify plasmid incompatibility groups,
replicon types, and plasmid sequence types — entirely client-side with no data
leaving your machine.

All computation runs client-side via WebAssembly.

## Problem

Plasmid typing is critical for understanding AMR gene mobility. Resistance
genes are often carried on plasmids, and knowing which plasmid type carries
which resistance gene is essential for epidemiological tracking and outbreak
investigation.

Currently, plasmid typing requires command-line tools (PlasmidFinder, MOB-
suite, pMLST from CGE) or uploading data to web services (CGE PlasmidFinder
web). There is no client-side browser tool for plasmid typing, despite the
approach being identical to chromosomal MLST — align against allele databases
and look up sequence types.

pMLSTx follows the same architecture as MLSTx (which already exists in the
GenomicX ecosystem for chromosomal MLST) but targets plasmid-specific schemes
and replicon databases.

## Core Concept

```
FASTA (assembled contigs)
    |
    ├──> [minimap2 — align contigs vs. PlasmidFinder replicon DB]
    |       Identify replicon types (Inc groups)
    |
    ├──> [minimap2 — align contigs vs. pMLST allele DB]
    |       Identify allele profiles for detected replicons
    |
    └──> [lookup — allele profile → sequence type]
            Determine plasmid ST from allele combination

    |
    v
Report (replicon types, pMLST profiles, sequence types)
```

## Target Users

- **Infection control teams** tracking plasmid-mediated AMR spread
- **Public health labs** doing plasmid surveillance (e.g., carbapenemase-
  carrying plasmids)
- **Researchers** studying horizontal gene transfer and plasmid epidemiology
- **Students** learning about plasmid biology and typing schemes

## Background: Plasmid Typing

### What is pMLST?

Plasmid MLST (pMLST) applies the same multi-locus sequence typing principle
as chromosomal MLST, but to plasmid-borne loci. Each pMLST scheme targets a
specific plasmid incompatibility group and uses 1–5 loci to define sequence
types.

### Incompatibility Groups

Plasmids are classified by incompatibility (Inc) groups — plasmids in the same
Inc group cannot stably coexist in the same cell. Major groups in
Enterobacteriaceae:

- **IncF** (IncFIA, IncFIB, IncFII) — most common in E. coli / Klebsiella,
  often carry ESBL genes
- **IncI1** — common carrier of ESBL/AmpC genes
- **IncN** — associated with carbapenemase genes
- **IncA/C** — broad host range, multi-drug resistance
- **IncHI1/HI2** — large plasmids in Salmonella, carry multi-drug resistance
- **IncX** (IncX1–X4) — increasingly associated with carbapenemases (NDM,
  OXA-48)

### Relationship to PlasmidFinder

PlasmidFinder detects replicon sequences (Inc group markers) via BLAST against
a curated database. pMLST goes one step further: once a replicon is detected,
pMLST types its specific loci to assign a sequence type. They are
complementary:

1. **PlasmidFinder** → "This contig carries an IncFII replicon"
2. **pMLST** → "This IncFII replicon is pMLST type ST1 (alleles: repA 1,
   sopB 2)"

## Pipeline Detail

### Step 1: Replicon Detection (PlasmidFinder approach)

- Align user assembly against the PlasmidFinder replicon database using
  minimap2
- Database: curated FASTA of replicon marker sequences (~200 sequences, <1 MB)
- Thresholds: identity >= 80%, coverage >= 60% (PlasmidFinder defaults)
- Output: list of detected Inc groups with identity and coverage scores

### Step 2: pMLST Allele Calling

For each detected replicon type that has an associated pMLST scheme:

- Align assembly against the pMLST allele database for that scheme using
  minimap2
- Each scheme has 1–5 loci, each with a set of known alleles
- Find the best-matching allele for each locus:
  - **Exact match** (100% identity, 100% coverage) → known allele number
  - **Novel allele** (>= 95% identity but not exact) → flag as "novel"
  - **No hit** (< 80% identity or coverage) → flag as "missing"

### Step 3: Sequence Type Lookup

- Combine allele numbers into an allele profile
- Look up the profile in the pMLST scheme's ST definition table
- If all alleles are exact matches and the profile exists → report the ST
- If any allele is novel or the profile doesn't exist → report "unknown ST"
  with the partial profile

### Available pMLST Schemes

Schemes are maintained on PubMLST. Current schemes include:

| Scheme    | Inc group | # loci | # STs (approx) |
| --------- | --------- | ------ | --------------- |
| IncF RST  | IncF      | 3 (FIA, FIB, FII replicon types) | ~1,500 |
| IncI1     | IncI1     | 5      | ~300            |
| IncN      | IncN      | 2      | ~50             |
| IncHI1    | IncHI1    | 2      | ~50             |
| IncHI2    | IncHI2    | 3      | ~100            |
| IncA/C    | IncA/C    | 3      | ~50             |

Total database size: allele FASTAs + ST profiles < 5 MB compressed.

## User Interface

### Upload Panel
- Drag-and-drop for FASTA assemblies (.fasta, .fa, .fna, .gz)
- Single or batch upload

### Results — Single Sample

**Replicon Summary:**
- Detected Inc groups shown as badges (e.g., "IncFIA", "IncFIB", "IncI1")
- Confidence indicator (exact / partial / low coverage)

**pMLST Profiles:**
- Table: one row per detected scheme
- Columns: Scheme, Allele 1, Allele 2, ..., Sequence Type
- Colour coding: exact matches (green), novel alleles (amber), missing (red)
- Expandable detail: identity/coverage scores per allele

**Context (informational):**
- Brief description of each detected Inc group and its epidemiological
  significance
- Link to PubMLST page for the scheme

### Results — Batch Mode
- Summary table: one row per sample, columns for detected replicons, pMLST
  types
- Filter by Inc group or ST
- Download as CSV/TSV

## Tech Stack

| Component       | Tool / Library                         |
| --------------- | -------------------------------------- |
| Alignment       | minimap2 (Biowasm)                     |
| WASM runtime    | Aioli (WebWorkers + PROXYFS)           |
| PAF parsing     | Custom JavaScript                      |
| Allele databases| PubMLST pMLST schemes (FASTA + TSV)   |
| Replicon DB     | PlasmidFinder database (FASTA)         |
| Frontend        | TypeScript, Vite, React                |
| Styling         | GenomicX design system (ronaQC tokens) |

## Performance Considerations

- **Database size**: Replicon DB (~200 sequences) + pMLST alleles (< 5 MB
  total). Tiny compared to Genetrax databases.
- **Alignment time**: A 5 MB assembly against < 5 MB of allele databases
  completes in < 3 seconds via minimap2 WASM.
- **Total time per sample**: < 5 seconds.
- **Memory**: < 200 MB total.
- **Batch**: 50 samples in < 5 minutes.

## Code Reuse from MLSTx

pMLSTx shares ~80% of its architecture with MLSTx:

- Same minimap2 alignment approach
- Same allele matching logic (exact / novel / missing)
- Same ST lookup mechanism
- Same UI patterns (allele profile table, colour-coded badges)

The key differences:
- Different databases (pMLST schemes vs. chromosomal MLST schemes)
- Additional replicon detection step (PlasmidFinder)
- Multiple schemes may apply to a single sample (a genome can carry multiple
  plasmid types)

Consider building pMLSTx as an extension of the MLSTx codebase, or
extracting shared typing logic into a common library.

## Relationship to Other GenomicX Tools

```
FASTA (assembly or Consensx output)
       |
       v
  [ pMLSTx ]  ──>  Plasmid replicon types + pMLST STs
       |
       ├──  Genetrax found AMR genes — pMLSTx tells you which plasmid carries them
       ├──  MLSTx typed the chromosome — pMLSTx types the plasmids
       └──  Specx confirmed the species — pMLSTx adds plasmid context
```

pMLSTx answers "what plasmids does this isolate carry?" — complementing
MLSTx's chromosomal typing and Genetrax's resistance gene detection.

## MVP Scope (v1)

1. User uploads one FASTA assembly
2. Detect replicon types via PlasmidFinder database + minimap2
3. Run pMLST allele calling for detected schemes
4. Look up sequence types
5. Display replicon badges + pMLST profile table
6. Download results as TSV

**Not in MVP:**
- Batch mode
- Inc group descriptions / epidemiological context
- Cross-referencing with Genetrax AMR results
- Custom scheme upload

## Prior Art

| Tool           | Scope              | Runs in browser? |
| -------------- | ------------------ | ---------------- |
| PlasmidFinder  | Replicon detection | Server-side (CGE)|
| pMLST (CGE)    | Plasmid MLST       | Server-side (CGE)|
| MOB-suite      | Plasmid typing     | No (Python)      |
| MLSTx          | Chromosomal MLST   | Yes (GenomicX)   |
| **pMLSTx**     | **Plasmid MLST**   | **Yes (minimap2 WASM)** |

pMLSTx brings plasmid typing to the browser using the same proven minimap2
WASM approach as MLSTx.

## Suggested apps.json Entry

```json
{
  "id": "pmlstx",
  "name": "pMLSTx",
  "tagline": "Plasmid MLST Typing",
  "description": "Type plasmid replicons using pMLST schemes entirely in the browser. Identify plasmid incompatibility groups and sequence types from bacterial genome assemblies with instant, privacy-preserving analysis.",
  "icon": "link",
  "tech": ["minimap2", "WebAssembly", "TypeScript", "Vite"],
  "features": [
    "Plasmid replicon typing",
    "Multiple pMLST scheme support",
    "Incompatibility group identification",
    "FASTA and gzipped input"
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
