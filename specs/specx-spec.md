# Specx — Browser-Based Bacterial Speciation & Assembly QC

## Overview

Specx identifies bacterial species and assesses assembly quality directly in
the browser. Upload genome assemblies (FASTA) to get rapid taxonomic
classification, contamination screening, and completeness metrics — no server
required.

All computation runs client-side via WebAssembly. No data leaves the user's
machine.

## Problem

Before running any downstream analysis (MLST, AMR, phylogenetics), you need
to know two things: **what species is this?** and **is the assembly any
good?** Currently this requires installing Mash, QUAST, or CheckM — tools
with complex dependencies that are inaccessible to many users. Existing web
services (BV-BRC, PathogenWatch) require uploading data to external servers,
which is a privacy concern for clinical and sensitive samples.

There is no browser-native tool that answers both questions from a single
FASTA upload.

## Core Concept

```
FASTA (assembled contigs)
    |
    ├──> [Assembly metrics — pure JS]
    |       N50, L50, GC%, total length, # contigs, largest contig
    |
    ├──> [Species ID — MinHash vs. RefSeq sketch]
    |       Top species hits with distance + identity scores
    |
    └──> [Contamination flag — multi-species heuristic]
            Flag if reads/contigs map to multiple species

    |
    v
Summary report (species, QC pass/fail, metrics)
```

## Target Users

- **Anyone starting a GenomicX analysis** — Specx is the natural first step
  before MLSTx, Genetrax, MashtreeWebx, etc.
- **Clinical labs** receiving assemblies from sequencing providers — quick
  sanity check before reporting
- **Students** learning what makes a good assembly
- **Surveillance teams** processing batches of assemblies and needing a fast
  species confirmation + QC gate

## Pipeline Detail

### Step 1: Assembly Metrics (Pure JavaScript)

No external tools needed. Parse the FASTA and compute:

| Metric          | Definition                                             |
| --------------- | ------------------------------------------------------ |
| Total length    | Sum of all contig lengths                              |
| # contigs       | Number of sequences in the FASTA                       |
| Largest contig  | Length of the longest sequence                          |
| N50             | Contig length where 50% of assembly is in longer contigs|
| L50             | Number of contigs needed to reach N50                  |
| GC content (%)  | (G+C) / total bases x 100                              |
| # contigs >=1kb | Count of contigs at least 1,000 bp                     |

These are trivial to compute in JS by iterating over FASTA records. No WASM
needed.

### Step 2: Species Identification (MinHash)

Two approaches:

**Option A — JavaScript MinHash (MVP)**
- Implement a lightweight MinHash sketch in JavaScript/TypeScript
- Ship a pre-built sketch database of ~5,000 NCBI RefSeq representative
  bacterial genomes (~10 MB compressed)
- Sketch the uploaded assembly, compute Jaccard distances against the database
- Return top 5 hits ranked by estimated ANI (Average Nucleotide Identity)
- Display: species name, ANI estimate, shared hashes, p-value

**Option B — Mash via custom WASM build (v2)**
- Compile Mash to WebAssembly (not currently in Biowasm, would need a custom
  build)
- Use `mash dist` against a full RefSeq sketch (~93 MB)
- More accurate, supports `mash screen` for read-level containment

The MVP JS approach is sufficient for species-level identification. Mash
distance correlates with ANI at r^2=0.99 for distances < 0.2.

### Step 3: Contamination Screening (Heuristic)

Browser-feasible approach using the MinHash results:

- If the top two species hits have similar distances (within 0.01) but are
  from distinct species/genera, flag as **potential contamination**
- Report: "Primary species: X (ANI 98.5%), Secondary species: Y (ANI 95.2%)
  — possible mixed sample"
- Not as rigorous as CheckM or ConFindr, but catches gross contamination
  (mixed species) without any heavy dependencies

### Step 4: QC Verdict

Apply configurable thresholds to produce a pass/warn/fail verdict:

| Check                | Pass          | Warn          | Fail          |
| -------------------- | ------------- | ------------- | ------------- |
| N50                  | >= 50 kb      | 20–50 kb      | < 20 kb       |
| Total length         | Within expected range for species | +/- 20% | > 20% off |
| # contigs            | < 200         | 200–500       | > 500         |
| GC content           | Within +/- 3% of species expected | +/- 5% | > 5% off |
| Species confidence   | ANI >= 95%    | 90–95%        | < 90%         |
| Contamination        | Single species| Low secondary  | Multi-species |

Thresholds are species-aware: expected genome size and GC% are looked up from
the RefSeq representative metadata once the species is identified.

## User Interface

### Upload Panel
- Drag-and-drop zone for FASTA files (.fasta, .fa, .fna, .gz)
- Support single or multiple assemblies (batch mode)
- File size and contig count preview on upload

### Results — Single Sample
- **Species card**: top hit species name, ANI score, confidence badge
  (high/medium/low)
- **Assembly metrics table**: N50, L50, GC%, total length, # contigs, largest
  contig
- **QC verdict banner**: PASS (green) / WARN (amber) / FAIL (red) with
  reasons
- **Contamination alert**: shown only if flagged
- **"Analyse with..."** buttons: MLSTx, Genetrax, MashtreeWebx

### Results — Batch Mode
- Summary table: one row per sample, columns for species, N50, # contigs, QC
  verdict
- Sortable and filterable
- Download as CSV/TSV
- Click any row to expand to single-sample detail view

## Tech Stack

| Component       | Tool / Library                              |
| --------------- | ------------------------------------------- |
| Assembly metrics| Pure JavaScript (no WASM needed)            |
| Species ID      | Custom JS MinHash (MVP) / Mash WASM (v2)   |
| Sketch database | Pre-built from RefSeq representatives       |
| Frontend        | TypeScript, Vite, React                     |
| Styling         | GenomicX design system (ronaQC tokens)      |

## Performance Considerations

- **FASTA parsing + metrics**: Near-instant for typical assemblies (< 10 MB).
  Pure JS, single pass.
- **MinHash sketching**: The bottleneck. Sketching a 5 MB assembly with k=21,
  s=1000 in JS should take < 5 seconds. Comparing against 5,000 reference
  sketches: < 2 seconds (distance computation is just set intersection).
- **Sketch database**: ~10 MB compressed, fetched once and cached in
  IndexedDB for repeat use.
- **Batch mode**: Process samples sequentially via Web Worker to keep UI
  responsive. Each sample takes ~5–10 seconds.
- **Memory**: Minimal. FASTA + sketch database fit comfortably in < 200 MB.

## Species Reference Database (Offline, Pre-Build)

Built as a build step, shipped with the app:

1. Download NCBI RefSeq representative prokaryotic genomes (assembly_summary)
2. Select one genome per species (~5,000–8,000 species)
3. Compute MinHash sketch (k=21, s=1000) for each
4. Store as a compact binary or JSON: { species, accession, sketch, expected_size, expected_gc }
5. Compress and host on CDN

Updates: rebuild quarterly from RefSeq, versioned.

## Relationship to Other GenomicX Tools

Specx is the **QC gate** — the first tool users should run:

```
FASTA (user's assembly)
       |
       v
  [ Specx ]  ──>  Species ID + QC verdict
       |
       ├──>  MLSTx       (now knows which scheme to use)
       ├──>  pMLSTx      (plasmid typing)
       ├──>  Genetrax    (AMR + virulence screening)
       ├──>  MashtreeWebx (distance tree)
       └──>  BRIGx       (circular comparison)
```

Specx output (species ID) directly informs scheme selection in MLSTx and
database selection in Genetrax.

## MVP Scope (v1)

1. User uploads one or more FASTA assemblies
2. Compute assembly metrics (N50, L50, GC%, contigs, total length)
3. Species identification via JS MinHash against RefSeq sketch
4. QC verdict (pass/warn/fail) with reasons
5. Download results as CSV

**Not in MVP:**
- Contamination screening
- Batch summary visualization
- Cross-tool linking
- Mash WASM build

## Prior Art

| Tool           | Scope              | Runs in browser? |
| -------------- | ------------------ | ---------------- |
| QUAST          | Assembly QC        | No (Python)      |
| CheckM / CheckM2 | Completeness/contamination | No (Python + 15GB DB) |
| Mash / refseq_masher | Species ID  | No (CLI)         |
| BV-BRC         | Full analysis      | Server-side      |
| PathogenWatch  | Species + typing   | Server-side      |
| **Specx**      | **Species + QC**   | **Yes (JS + WASM)** |

No existing tool combines species ID and assembly QC in a client-side browser
application.

## Suggested apps.json Entry

```json
{
  "id": "specx",
  "name": "Specx",
  "tagline": "Rapid Speciation & Quality Control",
  "description": "Identify bacterial species and assess assembly quality directly in the browser. Upload genome assemblies to get rapid taxonomic classification, contamination screening, and completeness metrics — no server required.",
  "icon": "microscope",
  "tech": ["MinHash", "WebAssembly", "TypeScript", "Vite"],
  "features": [
    "Rapid species identification from assemblies",
    "Assembly quality and completeness checks",
    "Contamination screening",
    "Client-side processing with instant results"
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
