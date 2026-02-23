# Consensusx — Browser-Based Bacterial Hybrid Consensus Assembly

## Overview

Consensusx is a browser-native tool that generates consensus sequences from raw
sequencing reads using a hybrid approach: map reads to a best-matching
reference genome to produce a reference-based consensus, then de novo assemble
any unmapped reads using Sparrowhawk to capture accessory content (AMR genes,
plasmids, phage) not present in the reference. The combined output is a FASTA
"faux assembly" suitable for downstream analyses with other GenomicX tools
(MLSTx, Genetrax, pMLSTx, Specx, MashtreeWebx, BRIGx).

All computation runs client-side via WebAssembly. No data leaves the user's
machine.

## Problem

Most GenomicX tools require assembled genomes (FASTA) as input. Many users —
particularly in clinical labs, teaching settings, or resource-limited
environments — have raw sequencing reads (FASTQ) but lack the infrastructure
or expertise to run de novo assembly pipelines (SPAdes, Flye, etc.). De novo
assembly is also computationally expensive and poorly suited to in-browser
execution for bacterial genomes.

Reference-based consensus is a lightweight alternative: map reads to a
suitable reference, call the consensus, and produce a usable FASTA. However,
a pure reference-based approach misses any content not in the reference —
accessory genes, mobile elements, plasmids, and phage that are unique to the
isolate. Consensusx solves this with a hybrid strategy: reference-based
consensus for the core genome, plus de novo assembly of unmapped reads (using
Sparrowhawk, a Rust-based assembler compiled to WebAssembly) to capture the
accessory content. The result is a more complete faux assembly than either
approach alone.

## Core Concept

```
FASTQ (raw reads)
    |
    v  [species detection — quick Mash screen]
    |
    v  [select best-matching reference genome]
    |
    v  [minimap2 — map reads to reference]
    |
    v  [samtools sort + index]
    |
    ├──> mapped reads ──> [samtools consensus] ──> reference-based contigs
    |
    └──> unmapped reads ──> [Sparrowhawk de novo assembly] ──> accessory contigs
                                                                (plasmids, AMR,
                                                                 phage, etc.)
    |
    v  [merge reference contigs + accessory contigs]
    |
    v
FASTA (hybrid "faux assembly")
```

The user uploads FASTQ files. Consensusx auto-detects the species (via Mash
sketch against a pre-built database of RefSeq representative genomes), selects
the best-matching reference, maps reads, and calls a consensus for the
reference-covered portion. Reads that don't map to the reference — which
typically represent accessory genome content like plasmids, AMR cassettes, and
phage — are extracted and de novo assembled using Sparrowhawk (a Rust-based
assembler compiled to WebAssembly). The two sets of contigs are merged into a
single FASTA that users can feed directly into MLSTx, Genetrax, or any other
tool expecting an assembly.

## Target Users

- **Clinical microbiologists** who receive FASTQ from sequencing providers but
  don't run assembly pipelines
- **Students** learning genomics who want to go from reads to results without
  installing command-line tools
- **Researchers** who need a quick-and-dirty consensus for typing or screening
  without spinning up a full pipeline
- **Field / low-resource labs** with a laptop and a browser but no
  bioinformatics server

## Pipeline Detail

### Step 1: Species Detection (Mash screen)

- Pre-compute Mash sketches for NCBI RefSeq representative prokaryotic
  genomes (~12,000 assemblies covering major bacterial species)
- Ship a compressed sketch database with the app (Mash sketch databases
  compress well — a representative set can be <100 MB)
- On FASTQ upload, sketch the reads and screen against the database
- Return top hits ranked by identity/shared hashes
- Auto-select the best match, or let the user pick from top candidates
- If the user already knows the species, allow manual selection to skip this
  step

### Step 2: Pan-Genome Reference Selection

Two approaches (start with Option A, evolve to Option B):

**Option A — Representative genome (MVP)**
Use the single best-matching RefSeq representative genome as the reference.
Simple, fast, and sufficient for core genome analyses like MLST. The consensus
will miss accessory genes not in that particular reference.

**Option B — Non-redundant pan-genome reference (v2)**
For supported species, ship a pre-built non-redundant pan-genome reference:
- Constructed offline using Panaroo or PPanGGoLiN from all RefSeq genomes of
  that species
- Deduplicated: one representative sequence per gene cluster
- Concatenated into a single multi-FASTA reference (core + accessory genes)
- Pre-indexed for minimap2

This captures accessory genome content (AMR genes, virulence factors, plasmid
loci) that a single reference would miss, making the consensus more useful for
tools like Genetrax and pMLSTx.

**Priority species for pan-genome references:**
- *Escherichia coli* (including Shigella)
- *Salmonella enterica*
- *Staphylococcus aureus*
- *Klebsiella pneumoniae*
- *Streptococcus pneumoniae*
- *Mycobacterium tuberculosis*
- *Listeria monocytogenes*
- *Campylobacter jejuni*
- *Neisseria meningitidis*
- *Pseudomonas aeruginosa*

### Step 3: Read Mapping (minimap2)

- Use minimap2 via Biowasm/Aioli
- Auto-detect read type from FASTQ headers or let user specify:
  - Illumina short reads: `-ax sr`
  - ONT long reads: `-ax map-ont`
  - PacBio HiFi: `-ax map-hifi`
- Output SAM piped to samtools for BAM conversion

### Step 4: BAM Processing (samtools)

- `samtools view -bS` — convert SAM to BAM
- `samtools sort` — coordinate-sort the BAM
- `samtools index` — index for downstream tools
- All via Biowasm virtual filesystem (PROXYFS), no disk I/O

### Step 5: Consensus Calling (Mapped Reads)

- `samtools mpileup -B -aa -d 0 -Q 20 | samtools consensus`
- Minimum depth threshold (default 10, user-configurable)
- Positions below threshold called as 'N'
- Minimum quality (default 20)
- Minimum frequency threshold for ambiguous bases (default 0.5)
- Output: reference-based consensus contigs (core genome)

### Step 6: Extract Unmapped Reads

- `samtools view -b -f 4` — extract unmapped reads from BAM
- `samtools fastq` — convert unmapped BAM back to FASTQ
- For paired-end data: extract both mates if either is unmapped (or only
  both-unmapped pairs, configurable)
- These reads represent content absent from the reference: plasmids, AMR gene
  cassettes, integrons, phage, novel genomic islands

### Step 7: De Novo Assembly of Unmapped Reads (Sparrowhawk)

- Assemble unmapped reads using **Sparrowhawk**
  (https://github.com/bacpop/sparrowhawk-web)
- Sparrowhawk is a Rust-based genome assembler compiled to WebAssembly — runs
  entirely in-browser
- Input: unmapped FASTQ (paired-end Illumina)
- Output: assembled contigs (FASTA) + de Bruijn graphs (GFA)
- Filter output contigs by minimum length (default 500 bp) to remove noise
- These "accessory contigs" capture content the reference-based consensus
  missed

**Why Sparrowhawk?**
- Already compiled to WebAssembly and proven in-browser
- Rust → WASM is clean and performant
- Designed for Illumina paired-end data
- 4 GB WASM memory limit is sufficient for unmapped reads (typically a small
  fraction of total reads)

### Step 8: Merge & Output

Combine reference-based consensus (Step 5) and accessory contigs (Step 7)
into a single multi-FASTA:

- Reference contigs are named by their reference chromosome/contig origin
- Accessory contigs are prefixed with `accessory_` to distinguish them
- Sort by length (longest first)

**Final outputs:**

- **Hybrid consensus FASTA** — primary output, downloadable. Contains both
  reference-based and de novo assembled contigs.
- **Mapping statistics** — total reads, mapped reads (%), unmapped reads (%),
  coverage depth, breadth of coverage, N content
- **Assembly statistics** — number of accessory contigs, total accessory
  length, N50 of accessory contigs
- **Coverage plot** — interactive per-position or per-contig depth
  visualization
- **Optional VCF** — variant calls relative to the reference

## User Interface

### Upload Panel
- Drag-and-drop zone for FASTQ files (supports .fastq, .fq, .fastq.gz,
  .fq.gz)
- Support paired-end (R1 + R2) and single-end reads
- File size indicator and read count estimate

### Species Selection
- Auto-detected species shown with confidence score
- Dropdown to override with manual species selection
- "Use custom reference" option — user uploads their own FASTA

### Settings (collapsible, sensible defaults)
- Read type: Auto / Illumina / ONT / PacBio HiFi
- Minimum depth: 10 (slider, 1–100)
- Minimum quality: 20 (slider, 0–40)
- Consensus caller: ivar (default) / samtools
- Include unmapped regions as N: yes/no

### Progress
- Step-by-step progress bar: Detecting species → Mapping reads → Sorting →
  Calling consensus → Assembling unmapped reads → Merging
- Real-time stats: reads processed, % mapped, % unmapped, estimated time
  remaining

### Results Panel
- Hybrid consensus FASTA preview (scrollable)
- Summary stats table:
  - Total reads / Mapped reads / % mapped / Unmapped reads / % unmapped
  - Mean coverage depth (reference-based contigs)
  - Breadth of coverage (% reference covered at >= min depth)
  - Consensus length / N content (%)
  - Accessory contigs: count, total length, N50
  - Reference used
- Coverage depth plot (interactive, zoomable)
- Download buttons: hybrid FASTA, reference-only FASTA, accessory-only FASTA,
  stats CSV, coverage plot PNG, VCF (optional)
- **"Analyse with..."** buttons linking directly to other GenomicX tools:
  - "Type with MLSTx" — opens MLSTx with the consensus pre-loaded
  - "Screen with Genetrax" — opens Genetrax with the consensus
  - "Build tree with MashtreeWebx" — opens MashtreeWebx

## Tech Stack

| Component        | Tool / Library                                |
| ---------------- | --------------------------------------------- |
| Read mapping     | minimap2 (Biowasm)                            |
| BAM processing   | samtools (Biowasm)                            |
| Consensus        | samtools consensus (Biowasm)                  |
| De novo assembly | Sparrowhawk (Rust → WASM, bacpop)             |
| Species detect   | Mash (Biowasm or custom JS MinHash)           |
| WASM runtime     | Aioli (manages WebWorkers + PROXYFS)          |
| Frontend         | TypeScript, Vite, React                       |
| Styling          | GenomicX design system (ronaQC tokens)        |
| Visualization    | D3.js or lightweight charting lib             |

## Performance Considerations

### Bacterial vs. Viral Scale

RonaQC handles SARS-CoV-2 (~30 kb genome). Bacterial genomes are 100–200x
larger (1–10 Mb). This changes the calculus:

| Factor             | Viral (RonaQC)       | Bacterial (Consensusx)     |
| ------------------ | -------------------- | ------------------------ |
| Reference size     | ~30 kb               | 1–10 Mb                  |
| Typical FASTQ size | 10–50 MB             | 200 MB – 1 GB+          |
| Mapping time       | seconds              | 1–5 minutes (estimated)  |
| Memory             | ~200 MB              | 1–2 GB (estimated)       |
| Coverage target    | 1000x+               | 30–100x                  |

### Mitigation Strategies

- **WORKERFS**: Mount user FASTQ files lazily via Biowasm's WORKERFS — avoids
  loading entire file into memory
- **Streaming**: Process reads in chunks if possible; minimap2 streams input
  naturally
- **Web Workers**: Run the full pipeline in a dedicated worker thread to keep
  the UI responsive
- **Pre-indexed references**: Ship minimap2 .mmi index files for supported
  species so indexing doesn't happen in-browser
- **Subsampling option**: For very deep sequencing, offer an option to
  subsample reads to e.g., 100x before mapping (reduces time, usually
  sufficient for consensus)
- **Progress feedback**: Long-running steps need clear progress indicators so
  users don't think the app has frozen

### Memory Budget

Target: works in a browser tab with 4 GB memory allocation. This means:

- FASTQ: mounted via WORKERFS, not in memory
- minimap2 index: ~50–100 MB for a bacterial genome
- BAM: kept in virtual FS, streamed during consensus
- Sparrowhawk assembly: the main memory consumer. Sparrowhawk-web currently
  has a 4 GB limit due to 32-bit WASM memory addressing. However, unmapped
  reads are typically only 5–20% of total reads, so the input to Sparrowhawk
  is much smaller than the full FASTQ.
- Consensus FASTA: tiny relative to input (<10 MB)

Should work on a modern laptop. Will not work on phones or tablets with <4 GB
available RAM. Sparrowhawk's 4 GB WASM limit is the binding constraint.

## Pan-Genome Reference Construction (Offline, Pre-Build)

This happens outside the browser, as a build step:

1. For each priority species, download all RefSeq complete genomes
2. Run Panaroo (or PPanGGoLiN) to identify gene clusters
3. Pick one representative sequence per cluster (longest, or highest quality)
4. Concatenate into a single multi-FASTA reference
5. Build minimap2 index (.mmi)
6. Build Mash sketch
7. Package as a downloadable reference bundle

Reference bundles would be hosted on a CDN and fetched on first use per
species. Cached locally in IndexedDB for repeat use.

Estimated sizes per species:
- Pan-genome FASTA: 5–15 MB (compressed)
- minimap2 index: 50–100 MB
- Mash sketch: <1 MB

## Relationship to Other GenomicX Tools

Consensusx is designed as a **feeder tool** — its output is the input for
everything else:

```
  FASTQ (user's raw data)
         |
         v
    [ Consensusx ]  ──>  FASTA consensus
         |
         ├──>  MLSTx      (sequence typing)
         ├──>  pMLSTx     (plasmid typing)
         ├──>  Genetrax   (AMR + virulence)
         ├──>  Specx      (species confirmation + QC)
         ├──>  MashtreeWebx  (distance tree)
         └──>  BRIGx      (circular comparison)
```

This makes Consensusx a potential "landing page" for users who start with reads.
It could evolve into a lightweight pipeline runner: upload FASTQ, get typing +
resistance + tree results in one session.

## MVP Scope (v1)

For the initial release:

1. User uploads paired-end FASTQ (plain or gzipped)
2. User selects species from a dropdown (no auto-detection yet)
3. Map reads to the RefSeq representative genome for that species
4. Call consensus with samtools consensus (default parameters)
5. Extract unmapped reads, assemble with Sparrowhawk
6. Merge reference consensus + accessory contigs into hybrid FASTA
7. Display mapping stats + assembly summary
8. Download hybrid consensus FASTA

**Not in MVP:**
- Auto species detection via Mash
- Pan-genome references (Option B)
- VCF output
- Coverage plot
- Cross-tool linking ("Analyse with MLSTx")
- Long read support (ONT/PacBio)
- Single-end read support (Sparrowhawk requires paired-end)

## Future Directions

- **Auto-pipeline mode**: Upload FASTQ, get MLST + AMR + tree in one click
- **Batch processing**: Multiple samples, produce a summary table
- **Mixed-species detection**: Flag if reads map to multiple species
  (contamination)
- **Variant calling**: Full SNP/indel calling with annotation
- **Core SNP alignment**: Multi-sample consensus for phylogenetic analysis
  (browser-based Snippy equivalent)

## Prior Art

| Tool                  | Scope          | Runs in browser? | Captures accessory? |
| --------------------- | -------------- | ---------------- | ------------------- |
| Snippy                | Bacterial      | No (CLI)         | No (ref only)       |
| ivar                  | Viral/Bacterial| No (CLI)         | No (ref only)       |
| ViralWasm-Consensus   | Viral          | Yes (WebAssembly)| No (ref only)       |
| Sparrowhawk-web       | General        | Yes (Rust → WASM)| Yes (de novo only)  |
| Bactopia              | Bacterial      | No (Nextflow)    | Yes (full pipeline) |
| Galaxy                | General        | Server-side      | Yes (full pipeline) |
| **Consensusx**          | **Bacterial**  | **Yes (WebAssembly)** | **Yes (hybrid)**|

Consensusx is the first browser-native tool to combine reference-based consensus
with de novo assembly of unmapped reads. ViralWasm-Consensus (Niema Lab, 2024)
proved browser-based mapping + consensus works. Sparrowhawk-web (bacpop)
proved browser-based de novo assembly works. Consensusx chains them together
with automatic reference selection to produce a more complete faux assembly
than either approach alone.

## Suggested apps.json Entry

```json
{
  "id": "consensusx",
  "name": "Consensusx",
  "tagline": "Reference-Based Consensus Assembly",
  "description": "Generate bacterial consensus sequences from raw sequencing reads entirely in your browser. Map reads to a species-appropriate pan-genome reference and produce a FASTA suitable for typing, resistance screening, and phylogenetics — no assembly pipeline required.",
  "icon": "layers",
  "tech": ["minimap2", "samtools", "Sparrowhawk", "WebAssembly"],
  "features": [
    "Hybrid consensus: reference mapping + de novo assembly of unmapped reads",
    "Auto species detection and reference selection",
    "Captures accessory genome (plasmids, AMR cassettes, phage)",
    "Feeds directly into MLSTx, Genetrax, and more"
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
