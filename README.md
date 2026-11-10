# LLM Functional Interpretation of Differential Gene Expression

This repository contains the notebooks and input data files used for LLM-based biological interpretation of colorectal adenoma differential gene-expression signatures, with comparison against GO Biological Process and KEGG enrichment results.

It accompanies the manuscript:

> **Evaluation of Large Language Models for Functional Interpretation of Differentially Expressed Genes**

The goal of this workflow is not to replace GO or KEGG enrichment analysis, but to evaluate whether a large language model (LLM) can act as an assisting interpretive layer for summarizing gene-list signals into biological themes.

## Datasets

| Folder name in this repository | GEO accession | Role in the manuscript | Platform | Groups |
|---|---|---|---|---|
| `GSE8671` | [GSE8671](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE8671) | Discovery dataset | GPL570 | 32 normal mucosa vs 32 adenomas |
| `VALIDATION` | [GSE50114](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE50114) | Validation dataset | GPL6480 | 37 vs 9 samples |

Files named `VALIDATION` in this repository correspond to **GSE50114** in the manuscript.

## Gene selection

Differential expression was computed with GEO2R (limma). For GO/KEGG enrichment and LLM annotation, genes were first filtered to:

* **Up-regulated:** log2FC ≥ 1 and p < 0.05
* **Down-regulated:** log2FC ≤ −1 and p < 0.05

and the **50 genes in each direction with the smallest p-values** were selected (100 genes in total). The same rule was applied to both datasets, and GO/KEGG enrichment and the LLM received the same genes.

## Models and settings

* Models: **GPT-OSS-20B** and **GPT-OSS-120B**
* Access: NVIDIA's OpenAI-compatible hosted API (`https://integrate.api.nvidia.com/v1`); no local GPU inference
* Decoding: temperature 0; top-p and random seed left at provider defaults
* Each prompt was sent as a single user-role message (no system message)
* Notebooks were run in Google Colab

The exact prompt templates, model identifiers, decoding parameters, retry rules and number of runs are given in **Supplementary File S1** (`supplementary_tables/`).

### Reproducibility note

The hosted API no longer serves these models, and top-p and seed were not fixed, so the exact LLM outputs cannot be regenerated. For this reason, **all notebooks are provided with their outputs saved**, and PDF copies of the executed notebooks are included in `notebooks/`. The evaluation code (hallucination check, GO/KEGG comparison, Jaccard-type overlap) can still be run on the saved outputs or on new data.

### API key

The notebooks need an API key to call the model. Do **not** write the key into the notebook. Store it as an environment variable or a Colab secret and read it in the code.

## Repository structure

```text
data/
  GSE8671/      UP-50, DOWN-50 gene lists and GO/KEGG enrichment files (discovery)
  VALIDATION/   UP-50, DOWN-50 gene lists and GO/KEGG enrichment files (GSE50114)
notebooks/      Executed notebooks (.ipynb) and PDF copies, one per dataset and model
figures/        Figures 1–7 of the manuscript
supplementary_tables/
                LLM-predicted GO/KEGG profiles (discovery and validation),
                Supplementary File S1 (prompts and settings) and
                Supplementary Table S2 (literature-based biological plausibility)
README.md
requirements.txt
```

## Important Rule

Do **not** rename the input data files randomly.
The notebook depends on the dataset name inside the file names to automatically match the UP, DOWN, GO, and KEGG files.

## File Naming Rules

All files from the same dataset must use the exact same `DATASET_NAME`.

For example, if the dataset name is `GSE8671`, the required file names are:

```text
UP-50-GSE8671.xlsx
DOWN-50-GSE8671.xlsx
GO-up-GSE8671.txt
GO-down-GSE8671.txt
KEGG-up-GSE8671.txt
KEGG-down-GSE8671.txt
```

The gene-list files should follow this format:

```text
UP-50-DATASETNAME.xlsx
DOWN-50-DATASETNAME.xlsx
```

The GO/KEGG enrichment files should follow this format:

```text
GO-up-DATASETNAME.txt
GO-down-DATASETNAME.txt
KEGG-up-DATASETNAME.txt
KEGG-down-DATASETNAME.txt
```

## Dataset Name Rule

The dataset name can be any name chosen by the user. It does not have to be `GSE8671` or `VALIDATION`.

For example, the following dataset names are acceptable:

```text
GSE8671
VALIDATION
DATASET2
TEST
MYDATA
COLON_ADENOMA
```

The only requirement is that the same dataset name must be written exactly the same in all related files.

For example, if the chosen dataset name is `DATASET2`, the required files should be:

```text
UP-50-DATASET2.xlsx
DOWN-50-DATASET2.xlsx
GO-up-DATASET2.txt
GO-down-DATASET2.txt
KEGG-up-DATASET2.txt
KEGG-down-DATASET2.txt
```

The notebook detects the dataset name automatically from the UP and DOWN gene-list files.

To run another dataset, the user only needs to change these two file names at the beginning of the notebook:

```python
up_file = DATA_DIR / "UP-50-GSE8671.xlsx"
down_file = DATA_DIR / "DOWN-50-GSE8671.xlsx"
```

For another dataset:

```python
up_file = DATA_DIR / "UP-50-DATASET2.xlsx"
down_file = DATA_DIR / "DOWN-50-DATASET2.xlsx"
```

After that, the notebook will automatically search for the matching GO and KEGG files using the same dataset name.

Therefore, the code logic does not need to be changed. Only the first two input file names should be updated when switching datasets.

## Column Rules

The UP and DOWN gene-list files should contain a gene-symbol column. Recommended column name:

```text
Gene.symbol
```

The GO/KEGG enrichment files should contain columns for:

* enrichment term or pathway name
* enriched genes
* adjusted p-value

Common accepted column names include:

```text
Term
Description
Pathway
Name
Genes
Adjusted P-value
Adjusted_P_value
adj.P.Val
FDR
qvalue
```

## Main Workflow

The workflow includes:

1. Loading pre-filtered Top-50 upregulated and Top-50 downregulated gene lists.
2. Generating structured LLM-based biological interpretations.
3. Running self-consistency analysis.
4. Applying API-based best-of-k candidate selection.
5. Checking for hallucinated genes.
6. Comparing selected LLM themes with real GO/KEGG enrichment results.
7. Generating LLM-predicted GO-like and KEGG-like profiles.
8. Comparing predicted profiles with real Top-10 GO/KEGG enrichment results.
9. Applying theme stability analysis.
10. Applying GO/KEGG significance caution flags.

### Comparison rule (Match / Partial / Unsupported)

Each LLM theme or predicted term is compared with the GO/KEGG reference terms by shared genes and shared keywords:

* **Match:** the best-scoring reference term shares at least two genes and one keyword, and has adjusted p ≤ 0.10
* **Partial:** at least one shared gene or keyword, but not all Match criteria
* **Unsupported:** no shared genes or keywords

Significance labels: adjusted p < 0.05 significant; 0.05–0.10 suggestive; > 0.10 not significant. The full rule is described in the Methods of the manuscript.

## Main Outputs

When a notebook is run, it generates the following CSV and JSON files in the working directory. They are not stored separately in this repository; the results from the runs reported in the manuscript are saved inside the executed notebooks.

```text
input_DATASETNAME_top50_upregulated_genes.csv
input_DATASETNAME_top50_downregulated_genes.csv
llm_gene_reasoning_outputs_DATASETNAME_top50.json
real_go_kegg_categorical_comparison_DATASETNAME.csv
real_go_kegg_categorical_summary_DATASETNAME.csv
LLM_predicted_GO_KEGG_profiles_DATASETNAME.csv
LLM_predicted_vs_real_GO_KEGG_categorical_comparison_DATASETNAME.csv
LLM_predicted_vs_real_GO_KEGG_categorical_summary_DATASETNAME.csv
GO_upregulated_top10_terms_DATASETNAME.csv
GO_downregulated_top10_terms_DATASETNAME.csv
KEGG_upregulated_top10_pathways_DATASETNAME.csv
KEGG_downregulated_top10_pathways_DATASETNAME.csv
```

## Notes on Interpretation

LLM outputs should be interpreted as hypothesis-generating biological summaries.

They should not be treated as statistically validated enrichment results because the LLM does not calculate:

* p-values
* adjusted p-values
* enrichment ratios
* database-ranked enrichment statistics

GO and KEGG remain necessary for database-grounded and statistically supported enrichment analysis.

## Disclaimer

This project is intended for academic and exploratory research purposes.
The LLM-generated outputs are not clinical recommendations and should not be used as diagnostic or therapeutic evidence.
