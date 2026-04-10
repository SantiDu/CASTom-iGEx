# CASTom-iGEx Module 2 Workflow

Decision-tree DAG for `Software/model_prediction` to help choose the correct script path.

## Decision-tree DAG

```mermaid
flowchart TD
    A([Start inputs ready<br/>geno cov pheno phenoAnn]) --> B{Genotype harmonized<br/>with training reference}

    B -->|Yes| C[PriLer_predictGeneExp_run.R<br/>Predict gene expression]
    B -->|No / partial overlap| D[PriLer_predictGeneExp_smallerVariantSet_run.R<br/>Predict with intersected variants]

    C --> E{Sample size LE 10000}
    D --> E

    %% Small dataset branch
    E -->|Yes small dataset| S1[Tscore_PathScore_diff_run.R<br/>Compute T scores and optional pathway scores]
    S1 --> S1b{Need custom pathway DB?}
    S1b -->|Yes| S2[pathScore_customGeneList_run.R]
    S1b -->|No| S3[pheno_association_smallData_run.R<br/>Association T score and pathway]
    S2 --> S4[pheno_association_smallData_customPath_run.R<br/>Association: custom pathways]
    S3 --> S5{Multiple cohorts?}
    S4 --> S5
    S5 -->|No| Z1([Final small-data association output])
    S5 -->|Yes standard pathways| S6[pheno_association_metaAnalysis_run.R<br/>or pheno_association_metaAnalysis_noPathScore_run.R]
    S5 -->|Yes custom pathways| S7[pheno_association_customPath_metaAnalysis_run.R]
    S6 --> Z1
    S7 --> Z1

    %% Large dataset branch
    E -->|No large dataset| L0[Combine_filteredGeneExpr.sh<br/>Merge and filter split predicted expression]
    L0 --> L1[Tscore_splitGenes_run.R<br/>Split-gene T-score computation]
    L1 --> L2{Pathway type}
    L2 -->|Reactome/GO| L3[PathwayScores_splitGenes_run.R]
    L2 -->|Custom gene sets| L3b[PathwayScores_splitGenes_customGeneList_run.R]

    L3 --> L4[pheno_association_prepare_largeData_run.R<br/>Prepare split inputs/info]
    L3b --> L4
    L4 --> L5[pheno_association_tscore_largeData_run.R<br/>Test gene T-scores by split]
    L4 --> L6[pheno_association_pathscore_largeData_run.R<br/>Test pathway scores by split]
    L5 --> L7{Custom pathway combine?}
    L6 --> L7
    L7 -->|No| L8[pheno_association_combine_largeData_run.R]
    L7 -->|Yes| L9[pheno_association_combine_largeData_customPath_run.R]
    L8 --> Z2([Final large-data association output])
    L9 --> Z2
```

## Quick run checklist

- Confirm genotype/covariate/phenotype files follow naming and required columns from `README.md`.
- Choose prediction script:
  - harmonized variants -> `PriLer_predictGeneExp_run.R`
  - non-harmonized or intersected variants -> `PriLer_predictGeneExp_smallerVariantSet_run.R`
- Choose branch by cohort size:
  - small (`<= 10,000`) -> direct T-score/pathway + association scripts
  - large (`> 10,000`) -> split/prepare/test/combine scripts
- If using custom pathways, switch to the `*_customPath*` or `*_customGeneList*` scripts at the marked DAG nodes.
- If multiple cohorts are analyzed in small-data mode, run the matching meta-analysis script.

## Main outputs at a glance

- Predicted expression: `predictedExpression.txt.gz`
- T-scores: `predictedTscores.txt` (small) or split `predictedTscore_splitGenes*.RData` (large)
- Pathway scores: Reactome/GO or custom pathway score files
- Final association outputs: `pval_*` `.RData` files (small/large/custom/meta)
