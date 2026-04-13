# CASTom-iGEx Module 2 Workflow

Decision-tree DAG for `Software/model_prediction` to help choose the correct script path.

## Decision-tree DAG

```mermaid
flowchart TD
    A([Start inputs ready<br/>geno cov pheno phenoAnn]) --> A0[Preprocess and harmonize genotype input<br/>variant QC build and allele alignment]
    A0 --> B{Harmonization status}

    B -->|Yes| C[PriLer_predictGeneExp_run.R<br/>Predict gene expression]
    B -->|No partial overlap| D[PriLer_predictGeneExp_smallerVariantSet_run.R<br/>Predict with intersected POS REF ALT variants]

    C --> E{Sample size LE 10000}
    D --> E

    %% Small dataset branch
    E -->|Yes small dataset| S1[Tscore_PathScore_diff_run.R<br/>Compute T scores and optional Reactome GO scores]
    S1 --> S2[pheno_association_smallData_run.R<br/>Run T score association always<br/>and Reactome GO if available]
    S2 --> S2b{Need custom pathway DB}
    S2b -->|No| S4{Multiple cohorts}
    S2b -->|Yes| S3[pathScore_customGeneList_run.R<br/>Build custom pathway scores from T scores]
    S3 --> S3b[pheno_association_smallData_customPath_run.R<br/>Custom pathways needs prior T score association]
    S3b --> S4
    S4 -->|No| Z1([Final small data association output])
    S4 -->|Yes standard pathways| S5[pheno_association_metaAnalysis_run.R<br/>or pheno_association_metaAnalysis_noPathScore_run.R]
    S4 -->|Yes custom pathways| S6[pheno_association_customPath_metaAnalysis_run.R]
    S5 --> Z1
    S6 --> Z1

    %% Large dataset branch
    E -->|No large dataset| L0[Combine_filteredGeneExpr.sh<br/>Merge and filter split predicted expression]
    L0 --> L1[Tscore_splitGenes_run.R<br/>Split gene T score computation]
    L1 --> L2{Pathway scores to compute}
    L2 -->|Reactome GO default| L3[PathwayScores_splitGenes_run.R<br/>compute Reactome and GO scores]
    L2 -->|Custom gene sets| L3b[PathwayScores_splitGenes_customGeneList_run.R]
    L2 -->|None| L4
    L3 --> L4[pheno_association_prepare_largeData_run.R<br/>Prepare t score and pathway inputs]
    L3b --> L4
    L4 --> L5[pheno_association_tscore_largeData_run.R<br/>Run T score association always]
    L5 --> L6{Pathway association available}
    L6 -->|No| L9([Final large data t score only output])
    L6 -->|Reactome GO| L7[pheno_association_pathscore_largeData_run.R<br/>one pathway DB per run<br/>run Reactome and GO separately]
    L6 -->|Custom pathways| L7b[pheno_association_pathscore_largeData_run.R<br/>for custom pathway splits]
    L7 --> L8[pheno_association_combine_largeData_run.R]
    L7b --> L8b[pheno_association_combine_largeData_customPath_run.R]
    L8 --> Z2([Final large data association output])
    L8b --> Z2
    Z2 --> M1{Need cohort meta analysis}
    M1 -->|No| M0([Stop])
    M1 -->|Yes T score plus Reactome GO| M2[pheno_association_metaAnalysis_run.R]
    M1 -->|Yes T score only| M3[pheno_association_metaAnalysis_noPathScore_run.R]
    M1 -->|Yes custom pathways| M4[pheno_association_customPath_metaAnalysis_run.R]
```

## Quick run checklist

- Confirm genotype/covariate/phenotype files follow naming and required columns from `README.md`.
- Choose prediction script:
  - harmonized variants -> `PriLer_predictGeneExp_run.R`
  - non-harmonized or partial overlap -> `PriLer_predictGeneExp_smallerVariantSet_run.R` (intersection by `POS REF ALT`, not full harmonization)
- Choose branch by cohort size:
  - small (`<= 10,000`) -> compute T scores then run `pheno_association_smallData_run.R` first
  - large (`> 10,000`) -> always run split T score association first, then optional pathway association
- Reactome and GO are the default pathway set when provided; custom pathways are optional and additive.
- In large-data pathway association, `pheno_association_pathscore_largeData_run.R` handles one pathway database per run.
- For Reactome plus GO in large-data mode, run pathway association separately for Reactome and GO, then combine.
- If using custom pathways in small-data mode, run `pheno_association_smallData_customPath_run.R` only after T-score association is complete.
- If using custom pathways in large-data mode, use the `*_customPath*` or `*_customGeneList*` scripts at the marked DAG nodes.
- If multiple cohorts are analyzed in small-data mode, run the matching meta-analysis script.
- For multiple cohorts in large-data mode, run the same meta-analysis scripts after each cohort has been combined into `pval_*` outputs.

## Main outputs at a glance

- Predicted expression: `predictedExpression.txt.gz`
- T-scores: `predictedTscores.txt` (small) or split `predictedTscore_splitGenes*.RData` (large)
- Pathway scores: Reactome/GO or custom pathway score files
- Final association outputs: `pval_*` `.RData` files (small/large/custom/meta)
