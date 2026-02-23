# Genetrax — Browser-Based AMR & Virulence Genotyping

## Overview

Genetrax screens bacterial genome assemblies for antimicrobial resistance
(AMR) genes and virulence factors directly in the browser. Upload a FASTA
assembly and get a detailed report of detected resistance genes, drug class
predictions, and virulence markers — no installs, no uploads to external
servers.

All computation runs client-side via WebAssembly. No data leaves the user's
machine.

## Problem

AMR and virulence genotyping is a routine step in bacterial genomics. The
standard tools (ABRicate, AMRFinderPlus, ResFinder, staramr) all require
command-line installation with complex dependencies (BLAST, HMMER, Python
environments). Web services exist (ResFinder web, CARD RGI web) but require
uploading sequence data to external servers — a privacy concern for clinical
isolates and pre-publication data.

There is no browser-native, client-side tool for AMR and virulence screening.

## Core Concept

```
FASTA (assembled contigs)
    |
    v  [minimap2 — align contigs against AMR/VF databases]
    |
    v  [parse PAF — extract identity, coverage, coordinates]
    |
    v  [filter — apply identity >= 80%, coverage >= 80%]
    |
    v  [annotate — join gene hits with drug class / phenotype metadata]
    |
    v
Report (detected genes, drug classes, virulence factors)
```

The architecture follows ABRicate's approach (align assembly against curated
gene databases, filter by thresholds) but replaces BLAST with minimap2, which
is already compiled to WebAssembly and available via Biowasm.

## Target Users

- **Clinical microbiologists** who need AMR profiles for isolates without
  sending data off-site
- **Public health labs** doing routine surveillance genotyping
- **Researchers** wanting a quick screen before deeper analysis
- **Students** learning about resistance mechanisms and gene detection

## Pipeline Detail

### Step 1: Database Loading

Ship pre-bundled gene databases as FASTA files + metadata TSV:

**AMR databases (user selects one or more):**

| Database     | Source         | Content                          | Size (approx) |
| ------------ | -------------- | -------------------------------- | -------------- |
| ResFinder    | CGE / DTU      | ~3,000+ acquired resistance genes| ~3 MB          |
| CARD         | McMaster       | Curated AMR gene sequences       | ~5 MB          |
| NCBI AMR     | NCBI           | Reference Gene Catalog           | ~4 MB          |
| ARG-ANNOT    | INSERM         | Older, broad coverage            | ~1 MB          |

**Virulence databases:**

| Database     | Source         | Content                          | Size (approx) |
| ------------ | -------------- | -------------------------------- | -------------- |
| VFDB (core)  | CAMS           | Experimentally confirmed VFs     | ~5 MB          |
| VirulenceFinder | CGE         | Species-specific VFs             | ~2 MB          |

Databases are fetched from CDN on first use, cached in IndexedDB. Total
footprint: ~10–20 MB compressed.

### Step 2: Alignment (minimap2)

- Use minimap2 via Biowasm/Aioli
- Mode: `-x asm5` (assembly-to-reference, <5% divergence) or `-c --cs` for
  CIGAR/CS tags
- Align user's contigs against each selected database FASTA
- Output: PAF format with identity and alignment coordinates

minimap2 is already proven in Biowasm (used by ViralWasm, MLSTx). No BLAST
compilation needed.

### Step 3: Hit Detection (JavaScript)

Parse PAF output and for each alignment:

1. Compute **% identity**: matching bases / alignment length (PAF cols 9/10)
2. Compute **% coverage**: alignment length on reference / reference gene
   length (PAF cols 6/7)
3. Apply thresholds:
   - Minimum identity: **80%** (default, user-configurable)
   - Minimum coverage: **80%** (default, user-configurable)
4. For overlapping hits to the same gene, keep the best (highest identity x
   coverage)

### Step 4: Annotation (JavaScript)

Join detected gene IDs against metadata files:

- **Gene name** (e.g., `blaTEM-1`, `mecA`, `vanA`)
- **Drug class** (e.g., beta-lactam, methicillin, vancomycin)
- **Resistance mechanism** (e.g., enzymatic inactivation, target modification)
- **Virulence category** (e.g., adhesin, toxin, capsule, iron uptake)
- **Contig location** (start, end, strand from PAF)

Metadata is shipped as TSV alongside each database FASTA.

### Step 5: Output

- **AMR summary table**: gene name, drug class, % identity, % coverage,
  contig, position, database
- **Virulence summary table**: gene name, category, % identity, % coverage,
  contig, position
- **Drug class summary**: grouped view — which drug classes have detected
  resistance genes
- **Downloadable reports**: TSV, CSV, JSON

## User Interface

### Upload Panel
- Drag-and-drop for FASTA assemblies (.fasta, .fa, .fna, .gz)
- Single or batch upload (multiple assemblies)

### Database Selection
- Checkboxes for AMR databases: ResFinder (default), CARD, NCBI AMR
- Checkboxes for virulence databases: VFDB (default), VirulenceFinder
- "Select all" / "Recommended" presets

### Settings (collapsible)
- Minimum identity: 80% (slider, 50–100%)
- Minimum coverage: 80% (slider, 50–100%)

### Results — Single Sample

**AMR Panel:**
- Drug class summary cards (e.g., "Beta-lactams: 3 genes detected") with
  colour coding
- Expandable detail table per drug class: gene, identity, coverage, contig
- Visual indicator for multi-drug resistance

**Virulence Panel:**
- Category summary (toxins, adhesins, capsule, etc.)
- Detail table: gene, category, identity, coverage, contig

**Contig Map (optional, v2):**
- Linear visualization of contigs with gene hits marked as coloured arrows

### Results — Batch Mode
- Summary table: one row per sample, columns for # AMR genes, # drug classes,
  # virulence genes, key resistance flags (e.g., ESBL, carbapenemase, MRSA)
- Heatmap: samples (rows) x drug classes (columns), colored by gene count
- Download full batch report as CSV/TSV

## Tech Stack

| Component      | Tool / Library                         |
| -------------- | -------------------------------------- |
| Alignment      | minimap2 (Biowasm)                     |
| WASM runtime   | Aioli (manages WebWorkers + PROXYFS)   |
| PAF parsing    | Custom JavaScript                      |
| Databases      | ABRicate-format FASTA + metadata TSV   |
| Frontend       | TypeScript, Vite, React                |
| Styling        | GenomicX design system (ronaQC tokens) |

## Performance Considerations

- **Database indexing**: minimap2 indexes the database FASTA on the fly.
  For small databases (~3–5 MB), this takes < 1 second.
- **Alignment**: A typical 5 MB bacterial assembly against a 3 MB gene
  database completes in < 5 seconds via minimap2 WASM.
- **Total time per sample**: ~5–10 seconds (dominated by alignment).
- **Memory**: minimap2 + databases + assembly fit in < 500 MB.
- **Batch mode**: Sequential processing in a Web Worker. 20 samples in
  ~2–3 minutes.

## Database Maintenance

Databases are versioned and updated independently of the app:

1. Pull latest FASTA + metadata from upstream sources (ResFinder GitHub, CARD
   downloads, VFDB downloads)
2. Reformat to consistent schema: gene_id, gene_name, drug_class (or
   virulence_category), accession, description
3. Compress and push to CDN with version tag
4. App checks for updates on load, prompts user to refresh if new version
   available
5. Cached in IndexedDB for offline use

All source databases are open-access / open-source.

## Relationship to Other GenomicX Tools

```
FASTA (assembly or Consensx output)
       |
       v
  [ Genetrax ]  ──>  AMR + virulence report
       |
       ├──  Specx told us the species (informs database choice)
       ├──  MLSTx told us the sequence type (epidemiological context)
       └──  MashtreeWebx places it in a phylogenetic context
```

Genetrax is the **resistance + virulence screening step** in the GenomicX
workflow. It consumes assemblies (or Consensx output) and benefits from
species context provided by Specx.

## MVP Scope (v1)

1. User uploads one FASTA assembly
2. Align against ResFinder (AMR) and VFDB core (virulence) using minimap2
3. Filter hits by identity >= 80% and coverage >= 80%
4. Display results as two tables (AMR genes, virulence factors)
5. Download results as TSV

**Not in MVP:**
- Multiple database selection
- Batch mode
- Drug class summary cards / heatmap
- Contig map visualization
- Configurable thresholds (use 80/80 defaults)

## Prior Art

| Tool           | Scope              | Runs in browser? |
| -------------- | ------------------ | ---------------- |
| ABRicate       | AMR + VF (BLAST)   | No (Perl + BLAST)|
| AMRFinderPlus  | AMR (BLAST + HMM)  | No (C++ + BLAST) |
| ResFinder web  | AMR                | Server-side      |
| CARD RGI web   | AMR                | Server-side      |
| staramr        | AMR + VF           | No (Python)      |
| PathogenWatch  | AMR                | Server-side      |
| **Genetrax**   | **AMR + VF**       | **Yes (minimap2 WASM)** |

Genetrax is the first client-side, browser-native AMR + virulence screening
tool. It uses minimap2 (already WASM-compiled) instead of BLAST, following
the architecture proven by ViralWasm and MLSTx.

## Suggested apps.json Entry

```json
{
  "id": "genetrax",
  "name": "Genetrax",
  "tagline": "AMR & Virulence Genotyping",
  "description": "Screen bacterial genomes for antimicrobial resistance genes and virulence factors in your browser. Upload assemblies to detect clinically relevant genetic markers with detailed annotation and exportable reports.",
  "icon": "zap",
  "tech": ["minimap2", "WebAssembly", "TypeScript", "Vite"],
  "features": [
    "AMR gene detection from assemblies",
    "Virulence factor screening",
    "Detailed gene annotations and reports",
    "CSV and JSON export"
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
