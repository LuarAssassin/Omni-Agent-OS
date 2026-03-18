# OpenClaw Medical Skills — Code & Structure Index

> **Purpose**: Comprehensive mapping of all 869 skills to their categories, file locations, and key attributes.

---

## I. Infrastructure Files

### Core Scripts

| File | Purpose | Lines | Language |
|------|---------|-------|----------|
| `scripts/validate_skill.py` | YAML frontmatter validation | 86 | Python |
| `.github/workflows/validate_skill.yml` | CI/CD pipeline | 126 | YAML |

### Documentation

| File | Purpose | Size |
|------|---------|------|
| `README.md` | Main documentation (English) | 175 KB |
| `README_zh.md` | Chinese translation | 162 KB |

---

## II. Skill Categories Overview

| Category | Count | Path Pattern | Description |
|----------|-------|--------------|-------------|
| General & Core | 10 | `skills/*` | Browser, search, document tools |
| Medical & Clinical | 119 | `skills/*` | Healthcare, clinical workflows |
| Scientific Databases | 43 | `skills/*` | Genomics, protein, drug databases |
| Bioinformatics | 239 | `skills/bio-*` | gptomics bio-* suite |
| Omics & Computational Biology | 59 | `skills/*` | Single-cell, proteomics, cheminformatics |
| ClawBio Pipelines | 21 | `skills/*` | Orchestration pipelines |
| BioOS Extended Suite | 285 | `skills/*` | Extended agent suite |
| Data Science & Tools | 93 | `skills/*` | Statistics, visualization, automation |
| **Total** | **869** | | |

---

## III. General & Core Skills (10)

| Skill | File | Has Examples | Frontmatter Fields |
|-------|------|--------------|-------------------|
| agent-browser | `skills/agent-browser/SKILL.md` | No | name, description, allowed-tools |
| find-skills | `skills/find-skills/SKILL.md` | No | name, description |
| multi-search-engine | `skills/multi-search-engine/SKILL.md` | No | name, description |
| wikipedia-search | `skills/wikipedia-search/SKILL.md` | No | name, description |
| deep-research | `skills/deep-research/SKILL.md` | No | name, description |
| pdf | `skills/pdf/SKILL.md` | No | name, description |
| docx | `skills/docx/SKILL.md` | No | name, description |
| xlsx | `skills/xlsx/SKILL.md` | No | name, description |
| pptx | `skills/pptx/SKILL.md` | No | name, description |
| doc-coauthoring | `skills/doc-coauthoring/SKILL.md` | No | name, description |

---

## IV. Medical & Clinical Skills (119)

### Literature & Research

| Skill | File | Description |
|-------|------|-------------|
| pubmed-search | `skills/pubmed-search/SKILL.md` | PubMed literature search |
| medical-research-toolkit | `skills/medical-research-toolkit/SKILL.md` | 14+ biomedical databases |
| medical-specialty-briefs | `skills/medical-specialty-briefs/SKILL.md` | Daily research briefs |
| bgpt-paper-search | `skills/bgpt-paper-search/SKILL.md` | Paper search with BGPT |
| arxiv-search | `skills/arxiv-search/SKILL.md` | arXiv preprint search |

### Clinical Documentation

| Skill | File | Description |
|-------|------|-------------|
| clinical-reports | `skills/clinical-reports/SKILL.md` | Comprehensive clinical documentation |
| soap-notes | `skills/soap-notes/SKILL.md` | SOAP note generation |
| discharge-summaries | `skills/discharge-summaries/SKILL.md` | Discharge summary creation |
| prior-auth-decisions | `skills/prior-auth-decisions/SKILL.md` | Prior authorization |

### Drug Research & Safety

| Skill | File | Description |
|-------|------|-------------|
| tooluniverse-drug-research | `skills/tooluniverse-drug-research/SKILL.md` | Comprehensive drug reports |
| tooluniverse-pharmacovigilance | `skills/tooluniverse-pharmacovigilance/SKILL.md` | Adverse event analysis |
| tooluniverse-drug-drug-interaction | `skills/tooluniverse-drug-drug-interaction/SKILL.md` | DDI prediction |
| clinicaltrials-database | `skills/clinicaltrials-database/SKILL.md` | Clinical trials lookup |
| fda-database | `skills/fda-database/SKILL.md` | FDA data queries |

### Oncology & Precision Medicine

| Skill | File | Description |
|-------|------|-------------|
| autonomous-oncology-agent | `skills/autonomous-oncology-agent/SKILL.md` | Cancer care automation |
| trialgpt-matching | `skills/trialgpt-matching/SKILL.md` | Trial matching |
| trial-eligibility-agent | `skills/trial-eligibility-agent/SKILL.md` | Eligibility screening |
| precision-oncology | `skills/precision-oncology/SKILL.md` | Precision oncology workflows |
| tumor-clonal-evolution-agent | `skills/tumor-clonal-evolution-agent/SKILL.md` | Clonal evolution analysis |
| tumor-heterogeneity-agent | `skills/tumor-heterogeneity-agent/SKILL.md` | Tumor heterogeneity |
| tme-immune-profiling-agent | `skills/tme-immune-profiling-agent/SKILL.md` | Immune profiling |

### Medical Imaging

| Skill | File | Description |
|-------|------|-------------|
| multimodal-medical-imaging | `skills/multimodal-medical-imaging/SKILL.md` | Multi-modal imaging |
| dicom-analyzer | `skills/dicom-analyzer/SKILL.md` | DICOM analysis |
| radiology-reporting | `skills/radiology-reporting/SKILL.md` | Radiology reports |
| pathology-analysis | `skills/pathology-analysis/SKILL.md` | Pathology analysis |

### Mental Health

| Skill | File | Description |
|-------|------|-------------|
| adhd-daily-planner | `skills/adhd-daily-planner/SKILL.md` | ADHD planning |
| mental-health-crisis | `skills/mental-health-crisis/SKILL.md` | Crisis intervention |
| therapy-session-notes | `skills/therapy-session-notes/SKILL.md` | Therapy documentation |

### Medical Devices & Regulatory

| Skill | File | Description |
|-------|------|-------------|
| meddev-iec-62304 | `skills/meddev-iec-62304/SKILL.md` | Medical device software lifecycle |
| meddev-iso-14971 | `skills/meddev-iso-14971/SKILL.md` | Risk management |
| meddev-fda-510k | `skills/meddev-fda-510k/SKILL.md` | FDA 510(k) submissions |
| meddev-ce-mark | `skills/meddev-ce-mark/SKILL.md` | CE marking process |
| hipaa-compliance | `skills/hipaa-compliance/SKILL.md` | HIPAA compliance |

### Health & Wellness

| Skill | File | Description |
|-------|------|-------------|
| weightloss-analyzer | `skills/weightloss-analyzer/SKILL.md` | Weight loss analysis |
| wearable-analysis-agent | `skills/wearable-analysis-agent/SKILL.md` | Wearable data analysis |
| travel-health-analyzer | `skills/travel-health-analyzer/SKILL.md` | Travel health |
| tcm-constitution-analyzer | `skills/tcm-constitution-analyzer/SKILL.md` | TCM constitution |

---

## V. Scientific Databases (43)

### Genomics & Variants

| Skill | File | Description |
|-------|------|-------------|
| bio-clinical-databases-clinvar-lookup | `skills/bio-clinical-databases-clinvar-lookup/SKILL.md` | ClinVar queries |
| bio-clinical-databases-dbsnp-queries | `skills/bio-clinical-databases-dbsnp-queries/SKILL.md` | dbSNP queries |
| bio-clinical-databases-gnomad-frequencies | `skills/bio-clinical-databases-gnomad-frequencies/SKILL.md` | gnomAD frequencies |
| bio-clinical-databases-variant-prioritization | `skills/bio-clinical-databases-variant-prioritization/SKILL.md` | Variant prioritization |
| dbsnp-database | `skills/dbsnp-database/SKILL.md` | dbSNP database |
| gnomad-database | `skills/gnomad-database/SKILL.md` | gnomAD database |

### Proteins & Pathways

| Skill | File | Description |
|-------|------|-------------|
| uniprot-database | `skills/uniprot-database/SKILL.md` | UniProt queries |
| alphafold-database | `skills/alphafold-database/SKILL.md` | AlphaFold structures |
| pdb-database | `skills/pdb-database/SKILL.md` | Protein Data Bank |
| bindingdb-database | `skills/bindingdb-database/SKILL.md` | BindingDB |
| zinc-database | `skills/zinc-database/SKILL.md` | ZINC database |
| chembl-database | `skills/chembl-database/SKILL.md` | ChEMBL |
| drugbank-database | `skills/drugbank-database/SKILL.md` | DrugBank |
| kegg-pathways | `skills/kegg-pathways/SKILL.md` | KEGG pathways |
| reactome-pathways | `skills/reactome-pathways/SKILL.md` | Reactome |

### Drugs & Chemicals

| Skill | File | Description |
|-------|------|-------------|
| pubchem-database | `skills/pubchem-database/SKILL.md` | PubChem |
| chemspider-database | `skills/chemspider-database/SKILL.md` | ChemSpider |
| side-effect-database | `skills/side-effect-database/SKILL.md` | Side effect data |

### Cancer Genomics

| Skill | File | Description |
|-------|------|-------------|
| tcga-preprocessing | `skills/tcga-preprocessing/SKILL.md` | TCGA data |
| ccle-database | `skills/ccle-database/SKILL.md` | CCLE cell lines |
| cosmic-database | `skills/cosmic-database/SKILL.md` | COSMIC mutations |
| cbioportal-database | `skills/cbioportal-database/SKILL.md` | cBioPortal |

---

## VI. Bioinformatics Skills (239)

### Alignment & Sequence

| Skill | File | Description |
|-------|------|-------------|
| bio-alignment-io | `skills/bio-alignment-io/SKILL.md` | Alignment file I/O |
| bio-alignment-pairwise | `skills/bio-alignment-pairwise/SKILL.md` | Pairwise alignment |
| bio-alignment-msa-parsing | `skills/bio-alignment-msa-parsing/SKILL.md` | MSA parsing |
| bio-alignment-msa-statistics | `skills/bio-alignment-msa-statistics/SKILL.md` | MSA statistics |
| bio-alignment-filtering | `skills/bio-alignment-filtering/SKILL.md` | Alignment filtering |
| bio-alignment-sorting | `skills/bio-alignment-sorting/SKILL.md` | Alignment sorting |
| bio-alignment-indexing | `skills/bio-alignment-indexing/SKILL.md` | Alignment indexing |
| bio-alignment-validation | `skills/bio-alignment-validation/SKILL.md` | Alignment validation |
| bio-alignment-files-bam-statistics | `skills/bio-alignment-files-bam-statistics/SKILL.md` | BAM statistics |

### Variant Analysis

| Skill | File | Description |
|-------|------|-------------|
| bio-variant-calling-gatk | `skills/bio-variant-calling-gatk/SKILL.md` | GATK variant calling |
| bio-variant-calling-freebayes | `skills/bio-variant-calling-freebayes/SKILL.md` | FreeBayes |
| bio-variant-calling-bcftools | `skills/bio-variant-calling-bcftools/SKILL.md` | bcftools |
| bio-variant-annotation-vep | `skills/bio-variant-annotation-vep/SKILL.md` | VEP annotation |
| bio-variant-annotation-snpEff | `skills/bio-variant-annotation-snpEff/SKILL.md` | snpEff |
| bio-variant-annotation-annovar | `skills/bio-variant-annotation-annovar/SKILL.md` | ANNOVAR |
| bio-variant-filtering-vqsr | `skills/bio-variant-filtering-vqsr/SKILL.md` | VQSR filtering |
| bio-variant-filtering-hard-filtering | `skills/bio-variant-filtering-hard-filtering/SKILL.md` | Hard filtering |
| vcf-annotator | `skills/vcf-annotator/SKILL.md` | VCF annotation |
| variant-interpretation-acmg | `skills/variant-interpretation-acmg/SKILL.md` | ACMG classification |
| varcadd-pathogenicity | `skills/varcadd-pathogenicity/SKILL.md` | Pathogenicity prediction |

### RNA-seq & Transcriptomics

| Skill | File | Description |
|-------|------|-------------|
| bio-rnaseq-qc-fastqc | `skills/bio-rnaseq-qc-fastqc/SKILL.md` | FastQC |
| bio-rnaseq-qc-multiqc | `skills/bio-rnaseq-qc-multiqc/SKILL.md` | MultiQC |
| bio-rnaseq-alignment-star | `skills/bio-rnaseq-alignment-star/SKILL.md` | STAR alignment |
| bio-rnaseq-alignment-hisat2 | `skills/bio-rnaseq-alignment-hisat2/SKILL.md` | HISAT2 |
| bio-rnaseq-quantification-salmon | `skills/bio-rnaseq-quantification-salmon/SKILL.md` | Salmon |
| bio-rnaseq-quantification-rsem | `skills/bio-rnaseq-quantification-rsem/SKILL.md` | RSEM |
| bio-rnaseq-differential-expression | `skills/bio-rnaseq-differential-expression/SKILL.md` | DESeq2/edgeR |
| tooluniverse-rnaseq-deseq2 | `skills/tooluniverse-rnaseq-deseq2/SKILL.md` | DESeq2 analysis |
| bio-workflows-rnaseq-to-de | `skills/bio-workflows-rnaseq-to-de/SKILL.md` | Full pipeline |

### Single-Cell Analysis

| Skill | File | Description |
|-------|------|-------------|
| anndata | `skills/anndata/SKILL.md` | AnnData operations |
| scanpy-workflows | `skills/scanpy-workflows/SKILL.md` | Scanpy workflows |
| seurat-workflows | `skills/seurat-workflows/SKILL.md` | Seurat workflows |
| bio-single-cell-qc | `skills/bio-single-cell-qc/SKILL.md` | scRNA-seq QC |
| bio-single-cell-clustering | `skills/bio-single-cell-clustering/SKILL.md` | Clustering |
| bio-single-cell-marker-genes | `skills/bio-single-cell-marker-genes/SKILL.md` | Marker genes |
| bio-single-cell-trajectory | `skills/bio-single-cell-trajectory/SKILL.md` | Trajectory analysis |
| bio-single-cell-integration | `skills/bio-single-cell-integration/SKILL.md` | Batch integration |
| tooluniverse-single-cell | `skills/tooluniverse-single-cell/SKILL.md` | Single-cell suite |
| universal-single-cell-annotator | `skills/universal-single-cell-annotator/SKILL.md` | Cell type annotation |

### Epigenomics

| Skill | File | Description |
|-------|------|-------------|
| bio-atac-seq-atac-qc | `skills/bio-atac-seq-atac-qc/SKILL.md` | ATAC-seq QC |
| bio-atac-seq-atac-peak-calling | `skills/bio-atac-seq-atac-peak-calling/SKILL.md` | Peak calling |
| bio-atac-seq-differential-accessibility | `skills/bio-atac-seq-differential-accessibility/SKILL.md` | Differential accessibility |
| bio-atac-seq-motif-deviation | `skills/bio-atac-seq-motif-deviation/SKILL.md` | Motif analysis |
| bio-atac-seq-nucleosome-positioning | `skills/bio-atac-seq-nucleosome-positioning/SKILL.md` | Nucleosome positioning |
| bio-atac-seq-footprinting | `skills/bio-atac-seq-footprinting/SKILL.md` | Footprinting |
| bio-chipseq-qc | `skills/bio-chipseq-qc/SKILL.md` | ChIP-seq QC |
| bio-chipseq-peak-calling | `skills/bio-chipseq-peak-calling/SKILL.md` | ChIP-seq peaks |
| bio-chipseq-differential-binding | `skills/bio-chipseq-differential-binding/SKILL.md` | Differential binding |
| bio-chipseq-motif-analysis | `skills/bio-chipseq-motif-analysis/SKILL.md` | Motif analysis |

### Genome Assembly

| Skill | File | Description |
|-------|------|-------------|
| bio-genome-assembly-de-novo | `skills/bio-genome-assembly-de-novo/SKILL.md` | De novo assembly |
| bio-genome-assembly-scaffolding | `skills/bio-genome-assembly-scaffolding/SKILL.md` | Scaffolding |
| bio-genome-assembly-polishing | `skills/bio-genome-assembly-polishing/SKILL.md` | Assembly polishing |
| bio-basecalling | `skills/bio-basecalling/SKILL.md` | Nanopore basecalling |

### Metagenomics

| Skill | File | Description |
|-------|------|-------------|
| bio-metagenomics-qc | `skills/bio-metagenomics-qc/SKILL.md` | Metagenomics QC |
| bio-metagenomics-profiling | `skills/bio-metagenomics-profiling/SKILL.md` | Taxonomic profiling |
| bio-metagenomics-functional-analysis | `skills/bio-metagenomics-functional-analysis/SKILL.md` | Functional analysis |
| bio-metagenomics-binning | `skills/bio-metagenomics-binning/SKILL.md` | Metagenomic binning |

### Immunoinformatics

| Skill | File | Description |
|-------|------|-------------|
| bio-immunoinformatics-epitope-prediction | `skills/bio-immunoinformatics-epitope-prediction/SKILL.md` | Epitope prediction |
| bio-immunoinformatics-mhc-binding | `skills/bio-immunoinformatics-mhc-binding/SKILL.md` | MHC binding |
| tcr-repertoire-analysis-agent | `skills/tcr-repertoire-analysis-agent/SKILL.md` | TCR analysis |
| bcr-repertoire-analysis | `skills/bcr-repertoire-analysis/SKILL.md` | BCR analysis |

### CRISPR Screens

| Skill | File | Description |
|-------|------|-------------|
| bio-crispr-screens-library-design | `skills/bio-crispr-screens-library-design/SKILL.md` | Library design |
| bio-crispr-screens-mageck-analysis | `skills/bio-crispr-screens-mageck-analysis/SKILL.md` | MAGeCK |
| bio-crispr-screens-jacks-analysis | `skills/bio-crispr-screens-jacks-analysis/SKILL.md` | JACKS |
| bio-crispr-screens-hit-calling | `skills/bio-crispr-screens-hit-calling/SKILL.md` | Hit calling |
| bio-crispr-screens-base-editing-analysis | `skills/bio-crispr-screens-base-editing-analysis/SKILL.md` | Base editing |
| tooluniverse-crispr-screen-analysis | `skills/tooluniverse-crispr-screen-analysis/SKILL.md` | CRISPR analysis |

### CNV & Structural Variants

| Skill | File | Description |
|-------|------|-------------|
| bio-copy-number-cnvkit-analysis | `skills/bio-copy-number-cnvkit-analysis/SKILL.md` | CNVkit |
| bio-copy-number-gatk-cnv | `skills/bio-copy-number-gatk-cnv/SKILL.md` | GATK CNV |
| bio-copy-number-cnv-annotation | `skills/bio-copy-number-cnv-annotation/SKILL.md` | CNV annotation |
| tooluniverse-structural-variant-analysis | `skills/tooluniverse-structural-variant-analysis/SKILL.md` | SV analysis |

---

## VII. Omics & Computational Biology (59)

### Single-Cell & Spatial

| Skill | File | Description |
|-------|------|-------------|
| squidpy-spatial | `skills/squidpy-spatial/SKILL.md` | Spatial omics |
| scanpy-spatial | `skills/scanpy-spatial/SKILL.md` | Scanpy spatial |
| seurat-spatial | `skills/seurat-spatial/SKILL.md` | Seurat spatial |
| tooluniverse-spatial-transcriptomics | `skills/tooluniverse-spatial-transcriptomics/SKILL.md` | Spatial transcriptomics |
| tooluniverse-spatial-omics-analysis | `skills/tooluniverse-spatial-omics-analysis/SKILL.md` | Spatial analysis |

### Proteomics

| Skill | File | Description |
|-------|------|-------------|
| bio-proteomics-maxquant | `skills/bio-proteomics-maxquant/SKILL.md` | MaxQuant |
| bio-proteomics-dia-nn | `skills/bio-proteomics-dia-nn/SKILL.md` | DIA-NN |
| bio-proteomics-msstats | `skills/bio-proteomics-msstats/SKILL.md` | MSstats |
| tooluniverse-proteomics-analysis | `skills/tooluniverse-proteomics-analysis/SKILL.md` | Proteomics suite |

### Cheminformatics

| Skill | File | Description |
|-------|------|-------------|
| rdkit-cheminformatics | `skills/rdkit-cheminformatics/SKILL.md` | RDKit |
| openbabel-conversion | `skills/openbabel-conversion/SKILL.md` | Open Babel |
| bio-admet-prediction | `skills/bio-admet-prediction/SKILL.md` | ADMET prediction |

### Protein Structure & Design

| Skill | File | Description |
|-------|------|-------------|
| alphafold | `skills/alphafold/SKILL.md` | AlphaFold structure prediction |
| colabfold | `skills/colabfold/SKILL.md` | ColabFold |
| esmfold | `skills/esmfold/SKILL.md` | ESMFold |
| proteinmpnn | `skills/proteinmpnn/SKILL.md` | ProteinMPNN design |
| ligandmpnn | `skills/ligandmpnn/SKILL.md` | LigandMPNN |
| rf diffusion | `skills/rfdiffusion/SKILL.md` | RFdiffusion |
| chaidiffusion | `skills/chaidiffusion/SKILL.md` | Chai diffusion |
| boltz | `skills/boltz/SKILL.md` | Boltz structure prediction |
| bindcraft | `skills/bindcraft/SKILL.md` | Binder design |
| binder-design | `skills/binder-design/SKILL.md` | Binder design |
| antibody-design-agent | `skills/antibody-design-agent/SKILL.md` | Antibody design |
| tcr-pmhc-prediction-agent | `skills/tcr-pmhc-prediction-agent/SKILL.md` | TCR-pMHC prediction |
| protein-thermodynamics | `skills/protein-thermodynamics/SKILL.md` | Thermodynamics |
| protein-dynamics | `skills/protein-dynamics/SKILL.md` | Molecular dynamics |

### Phylogenetics

| Skill | File | Description |
|-------|------|-------------|
| bio-phylogenetics-tree-construction | `skills/bio-phylogenetics-tree-construction/SKILL.md` | Tree building |
| bio-phylogenetics-tree-visualization | `skills/bio-phylogenetics-tree-visualization/SKILL.md` | Tree viz |
| bio-phylogenetics-ancestral-reconstruction | `skills/bio-phylogenetics-ancestral-reconstruction/SKILL.md` | Ancestral states |
| tooluniverse-phylogenetics | `skills/tooluniverse-phylogenetics/SKILL.md` | Phylogenetics suite |

---

## VIII. ClawBio Pipelines (21)

| Skill | File | Description |
|-------|------|-------------|
| clawbio-scrna-pipeline | `skills/clawbio-scrna-pipeline/SKILL.md` | scRNA-seq pipeline |
| clawbio-gwas-pipeline | `skills/clawbio-gwas-pipeline/SKILL.md` | GWAS pipeline |
| clawbio-variant-pipeline | `skills/clawbio-variant-pipeline/SKILL.md` | Variant calling pipeline |
| clawbio-ancestry-pipeline | `skills/clawbio-ancestry-pipeline/SKILL.md` | Ancestry analysis |
| clawbio-structure-pipeline | `skills/clawbio-structure-pipeline/SKILL.md` | Structure prediction pipeline |

---

## IX. BioOS Extended Suite (285)

The BioOS Extended Suite contains extended agent implementations for oncology, immunology, clinical AI, and research infrastructure. Skills follow similar patterns to above but with enhanced capabilities.

Key patterns:
- `*-agent` suffix for autonomous agents
- `*-pipeline` suffix for workflow orchestration
- Integration with ToolUniverse APIs

---

## X. Data Science & Tools (93)

### Statistics & Analysis

| Skill | File | Description |
|-------|------|-------------|
| tooluniverse-statistical-modeling | `skills/tooluniverse-statistical-modeling/SKILL.md` | Statistical modeling |
| bayesian-optimizer | `skills/bayesian-optimizer/SKILL.md` | Bayesian optimization |
| tooluniverse-gwas-drug-discovery | `skills/tooluniverse-gwas-drug-discovery/SKILL.md` | GWAS for drug discovery |
| tooluniverse-gwas-finemapping | `skills/tooluniverse-gwas-finemapping/SKILL.md` | Fine-mapping |
| tooluniverse-gwas-trait-to-gene | `skills/tooluniverse-gwas-trait-to-gene/SKILL.md` | Trait-to-gene |

### Visualization

| Skill | File | Description |
|-------|------|-------------|
| bio-data-visualization-ggplot2-fundamentals | `skills/bio-data-visualization-ggplot2-fundamentals/SKILL.md` | ggplot2 |
| bio-data-visualization-heatmaps-clustering | `skills/bio-data-visualization-heatmaps-clustering/SKILL.md` | Heatmaps |
| bio-data-visualization-genome-browser-tracks | `skills/bio-data-visualization-genome-browser-tracks/SKILL.md` | Genome tracks |
| bio-data-visualization-interactive-visualization | `skills/bio-data-visualization-interactive-visualization/SKILL.md` | Interactive viz |
| bio-data-visualization-circos-plots | `skills/bio-data-visualization-circos-plots/SKILL.md` | Circos plots |

### Machine Learning

| Skill | File | Description |
|-------|------|-------------|
| bio-machine-learning-scvi-tools | `skills/bio-machine-learning-scvi-tools/SKILL.md` | scVI-tools |
| bio-machine-learning-celltypist | `skills/bio-machine-learning-celltypist/SKILL.md` | CellTypist |
| bio-machine-learning-atlas-mapping | `skills/bio-machine-learning-atlas-mapping/SKILL.md` | Atlas mapping |
| transformers | `skills/transformers/SKILL.md` | Hugging Face |
| torch-geometric | `skills/torch-geometric/SKILL.md` | PyTorch Geometric |
| torchdrug | `skills/torchdrug/SKILL.md` | TorchDrug |

### Python Libraries

| Skill | File | Description |
|-------|------|-------------|
| pandas | `skills/pandas/SKILL.md` | Data manipulation |
| numpy | `skills/numpy/SKILL.md` | Numerical computing |
| scipy | `skills/scipy/SKILL.md` | Scientific computing |
| matplotlib | `skills/matplotlib/SKILL.md` | Plotting |
| seaborn | `skills/seaborn/SKILL.md` | Statistical viz |
| umap-learn | `skills/umap-learn/SKILL.md` | UMAP |
| vaex | `skills/vaex/SKILL.md` | Large dataframes |
| zarr-python | `skills/zarr-python/SKILL.md` | Zarr storage |
| anndata | `skills/anndata/SKILL.md` | Annotated data |
| tiledbvcf | `skills/tiledbvcf/SKILL.md` | TileDB VCF |

---

## XI. Implementation File Index

### Python Examples by Category

```
skills/*/examples/*.py
├── bio-alignment-io/examples/
│   ├── read_alignment.py
│   ├── slice_alignment.py
│   └── convert_formats.py
├── bio-chipseq-qc/examples/
│   └── chipseq_qc.py
├── bio-restriction-enzyme-selection/examples/
│   ├── cloning_strategy.py
│   ├── golden_gate_check.py
│   └── select_enzymes.py
├── tooluniverse-single-cell/scripts/
│   ├── normalize_data.py
│   ├── qc_metrics.py
│   └── find_markers.py
└── [many more...]
```

### Shell Scripts by Category

```
skills/*/examples/*.sh
├── bio-basecalling/examples/
│   ├── basecall_pipeline.sh
│   └── dorado_basecall.sh
├── bio-crispr-screens-base-editing-analysis/examples/
│   └── base_editing_analysis.sh
├── bio-genome-assembly-scaffolding/examples/
│   └── yahs_scaffolding.sh
├── bio-clip-seq-clip-peak-calling/examples/
│   └── call_peaks.sh
└── [many more...]
```

### Multi-file Skills

Some skills have extensive implementations:

| Skill | Files | Description |
|-------|-------|-------------|
| multimodal-medical-imaging | `multimodal_agent.py`, `config/`, `utils/` | Complex agent |
| tooluniverse-gwas-study-explorer | `python_implementation.py`, `test_skill_comprehensive.py` | Full implementation |
| clinical-reports | `references/` directory | Extended documentation |
| anndata | `references/` directory with 6 markdown files | Comprehensive guides |

---

## XII. Frontmatter Patterns Index

### Minimal Pattern (Most Common)
```yaml
---
name: skill-name
description: Description of what this skill does
---
```

### Tool Declaration Pattern
```yaml
---
name: skill-name
description: Description
allowed-tools: Bash(agent-browser:*)
---
```

### Rich Metadata Pattern
```yaml
---
name: skill-name
description: Description
tool_type: python
primary_tool: Bio.AlignIO
license: MIT
category: bioinformatics
tags: [alignment, phylogenetics]
---
```

### Agent Pattern
```yaml
---
name: agent-name
description: Autonomous agent description
allowed-tools: [tool1, tool2]
biomodals_script: modal_script.py
---
```

---

## XIII. Quick Lookup

### By Prefix

| Prefix | Count | Description |
|--------|-------|-------------|
| `bio-` | 239 | Bioinformatics tools |
| `tooluniverse-` | ~50 | ToolUniverse integration |
| `meddev-` | ~10 | Medical device regulatory |
| `clinical-` | ~15 | Clinical workflows |
| `alphafold` | 2 | AlphaFold-related |
| `*-agent` | ~30 | Autonomous agents |
| `*-pipeline` | ~20 | Workflow pipelines |

### By Domain

| Domain | Representative Skills |
|--------|----------------------|
| Genomics | variant-calling, gwas, rnaseq, chipseq, atacseq |
| Proteomics | alphafold, proteinmpnn, maxquant, diann |
| Drug Discovery | bindcraft, chembl, drugbank, admet |
| Clinical | soap-notes, clinical-reports, trials |
| Imaging | dicom, pathology, multimodal-imaging |
| Statistics | deseq2, bayesian, statistical-modeling |

---

*Index generated from analysis of 869 skills.*
*Generated: 2026-03-17*
