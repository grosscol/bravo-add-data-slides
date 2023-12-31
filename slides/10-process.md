---
author: Colin Gross
title: Adding Data to Bravo
date: 2023-10-12
---

# Processing eQTL Data
- What is present.
- How to integrate with existing data.
- Making a pipeline.

## Data Structure
Wonderfully written readme describing the data and procees by which it was generated by Peter Orchard.

```md
==================
Freeze 1RNA README
==================
This is freeze 1RNA for the TOPMed RNA-seq data, containing cis-eQTL analysis results for six tissues:
* Whole blood (N = 6,602)
* Lung (N = 1,360)
* PBMC (N = 1,265)
* T cells (N = 368)
* Nasal epithelial (N = 359)
* Monocytes (N = 352)
...
   * Column descriptions:
      * phenotype_id: gene identifier
      * variant_id: genetic variant, in format {variant_chromosome}_{variant_position}_{variant_ref_allele}_{variant_alt_allele}
      * pip: SuSiE PIP (essentially, the probability the variant is a causal one for this eQTL signal)
      * af: frequency of the alt allele
      * cs_id: Credible set ID. cs_id + phenotype_id together uniquely identify a credible set. A credible set containing more than one genetic variant will span more than one line.
```

## Integration with Exisiting Data

incoming eqtl data (data/susie/PBMC.maf001.cs.txt):
```txt
phenotype_id    variant_id      pip     af      cs_id
ENSG00000151240.17      chr10_501718_G_T        0.98997444      0.78498024      1
ENSG00000107929.14      chr10_800029_T_C        0.016404094     0.3806324       1
ENSG00000107929.14      chr10_805438_T_G        0.18358973      0.38418972      1
```

existing gene data:
```json
{
        "chrom" : "1",
        "start" : 11869,
        "stop" : 14409,
        "gene_id" : "ENSG00000223972",
}

```

## Working Out Processing
- Remove trailing version numbers from Ensemble gene ids (column 1)
- Add tissue origin column (from filename) as last column

```sh
awk 'BEGIN {FS="\t"; OFS=FS}\
  (NR>1) { sub(/\\.[[:digit:]]+$/,"",$1); \
  print $0 OFS "${tissue_type}" }' ${tissue_tsv} \
    > ${tissue_type}.${analysis_type}.tsv
```

## Making a Nextflow Workflows
Nextflow V2 DSL organizes a workflow into smaller units of named workflows.

```nf
workflow {
  conditional_eqtl()
  susie_eqtl()
}
```

## Named Workflow
Named workflows are compositions of processes that are called like functions

```nf
workflow susie_eqtl {
  analysis_type = "susie"
  susie_tsv     = channel.fromPath("${params.susie_eqtl_glob}")
  susie_headers = channel.of(params.susie_fields).collect()
  fields        = params.susie_fields
  types         = params.susie_types

  validate_header(susie_tsv, susie_headers)
  munge_files(susie_tsv, analysis_type)
  merge_files(munge_files.out.collect(), 
              analysis_type, fields, types)
}
```

## Processes
Inputs and outputs with a defined script or program to run.

```nf
process munge_files {
  label "highcpu"

  input:
  file tissue_tsv
  val analysis_type

  output:
  file "*.tsv"

  shell:
  tissue_type = tissue_tsv.getFileName().toString().tokenize(".")[0]
  '''
  awk 'BEGIN {FS="\t"; OFS=FS}\
    (NR>1) { sub(/\\.[[:digit:]]+$/,"",$1); \
    print $0 OFS "!{tissue_type}" }' !{tissue_tsv} > !{tissue_type}.!{analysis_type}.tsv
  '''
}

```
