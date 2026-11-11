# LLM Functional Interpretation of Differential Gene Expression

Code and data accompanying the manuscript:

> **Evaluation of Large Language Models for Functional Interpretation of Differentially Expressed Genes**

This repository contains the notebooks, input gene lists, GO/KEGG enrichment files, figures and supplementary files used to evaluate large language models (LLMs) as an interpretive layer for colorectal adenoma differential gene-expression signatures, benchmarked against GO Biological Process and KEGG pathway enrichment.

The workflow is **not** intended to replace GO or KEGG enrichment analysis. It tests whether an LLM, guided by structured prompts and rule-based checks, can summarise gene-list signals into biological themes that are traceable to the input genes and comparable with enrichment results.

---

## Contents

1. [Datasets](#datasets)
2. [Gene selection](#gene-selection)
3. [Models and settings](#models-and-settings)
4. [Reproducibility and model availability](#reproducibility-and-model-availability)
5. [Repository structure](#repository-structure)
6. [Quick start](#quick-start)
7. [Input files](#input-files)
8. [Workflow](#workflow)
9. [Comparison rule](#comparison-rule-match--partial--unsupported)
10. [Outputs](#outputs)
11. [Interpretation and limitations](#interpretation-and-limitations)
12. [License, citation and disclaimer](#license-citation-and-disclaimer)

---

## Datasets

| Folder name in this repository | GEO accession | Role in the manuscript | Platform | Samples |
|---|---|---|---|---|
| `GSE8671` | [GSE8671](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE8671) | Discovery dataset | GPL570 | 32 normal mucosa, 32 adenomas |
| `VALIDATION` | [GSE50114](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE50114) | Validation dataset | GPL6480 | 37 vs 9 samples |

Files named `VALIDATION` in this repository correspond to **GSE50114** in the manuscript. The validation dataset was not used to change prompts or selection rules.

## Gene selection

Differential expression (normal mucosa vs adenoma) was computed with GEO2R (limma). For GO/KEGG enrichment and LLM annotation, genes were first filtered to:

* **Up-regulated:** log2FC ≥ 1 and p < 0.05
* **Down-regulated:** log2FC ≤ −1 and p < 0.05

The **50 genes in each direction with the smallest p-values** were then selected (100 genes per dataset). The same rule was applied to both datasets, and GO/KEGG enrichment and the LLM received exactly the same genes.

This fixed list favours the most significant genes among those with at least a two-fold change; the effect of list size and selection rule was not tested.

## Models and settings

| Setting | Value |
|---|---|
| Models | GPT-OSS-20B (primary), GPT-OSS-120B (comparison of model scale) |
| Access | NVIDIA's OpenAI-compatible hosted API (`https://integrate.api.nvidia.com/v1`); no local GPU inference |
| Environment | Python notebooks run in Google Colab |
| Temperature | 0 |
| Top-p, random seed | Left at provider defaults (not fixed) |
| Message format | Each prompt sent as a single user-role message; no system message |
| Run dates | GPT-OSS-20B: 12 June 2026; GPT-OSS-120B: 25 June 2026 |

The exact prompt templates, model identifiers, decoding parameters, retry rules, number of runs and recorded runtimes are given in **Supplementary File S1** (`supplementary_tables/`).

## Reproducibility and model availability

* **GPT-OSS-120B** has since been retired from the NVIDIA hosted API (end of life 3 September 2026).
* **GPT-OSS-20B** remained available on the API at the time of writing.
* Because top-p and random seed were not fixed, **exact outputs cannot be regenerated with either model**, even where the model is still served.

For this reason, the outputs of all four runs reported in the manuscript (2 datasets × 2 models) are provided as **PDF copies of the executed notebooks** in `notebooks/`. These PDFs are the record of the reported results.

The same notebook code is used for every run; only the input files and the model name change. To apply the workflow to new data, set the model name and API endpoint in the notebook to a model that is currently available. The evaluation steps (hallucination check, GO/KEGG comparison, Jaccard-type overlap) then run on the new model outputs; results from a different model or provider are not expected to match the reported values.

## Repository structure

```text
data/
  GSE8671/        UP-50 and DOWN-50 gene lists; GO-up, GO-down, KEGG-up, KEGG-down enrichment files (discovery)
  VALIDATION/     Same six files for the validation dataset (GSE50114)
notebooks/        Two notebook code files (.ipynb) and four PDF copies of the executed runs
                  (GSE8671 and GSE50114, each with GPT-OSS-20B and GPT-OSS-120B)
figures/          Figures 1–7 of the manuscript
supplementary_tables/
                  Supplementary File S1: prompts, settings and runtimes (PDF and Markdown)
                  Supplementary Table S2: literature-based biological plausibility assessment (Excel)
                  Supplementary GO/KEGG tables for the discovery (original) and validation datasets
LICENSE
README.md
requirements.txt
```

### Where each manuscript item comes from

| Manuscript item | Source in this repository |
|---|---|
| Input genes (Top-50 up/down) | `data/<dataset>/UP-50-*.xlsx`, `DOWN-50-*.xlsx` |
| GO/KEGG enrichment (Figures 3, 4, 6) | `data/<dataset>/GO-*.txt`, `KEGG-*.txt` |
| Selected themes, hallucination check, theme-level comparison (Table 3) | Executed notebooks (`notebooks/*.pdf`) |
| Profile-level comparison and Jaccard-type indices (Table 5, Figure 5) | Executed notebooks; supplementary GO/KEGG tables |
| Biological plausibility (Table 4) | `supplementary_tables/Supplementary_Table_S2_Biological_Plausibility.xlsx` |
| Prompts, settings and runtimes | `supplementary_tables/Supplementary_File_S1_Prompts_and_Settings.pdf` |

## Quick start

1. Clone the repository and install the dependencies:
   ```bash
   pip install -r requirements.txt
   ```
2. Open the notebook in Google Colab or Jupyter.
3. Provide your API key as an environment variable or a Colab secret, and make sure the notebook reads it from there. **Never write the key into the notebook or commit it to the repository.**
4. Set the model name to a model that is currently available (see [model availability](#reproducibility-and-model-availability)).
5. Check that `DATA_DIR` points to the folder of the dataset you want to run, and set the two input file names (see below).
6. Run all cells. Output files are written to the working directory.

## Input files

### Important rule

Do **not** rename the input data files randomly. The notebook uses the dataset name inside the file names to match the UP, DOWN, GO and KEGG files automatically.

### File naming

All files from the same dataset must use exactly the same `DATASETNAME`:

```text
UP-50-DATASETNAME.xlsx
DOWN-50-DATASETNAME.xlsx
GO-up-DATASETNAME.txt
GO-down-DATASETNAME.txt
KEGG-up-DATASETNAME.txt
KEGG-down-DATASETNAME.txt
```

For example, for `GSE8671`:

```text
UP-50-GSE8671.xlsx
DOWN-50-GSE8671.xlsx
GO-up-GSE8671.txt
GO-down-GSE8671.txt
KEGG-up-GSE8671.txt
KEGG-down-GSE8671.txt
```

The dataset name can be any name (for example `GSE8671`, `VALIDATION`, `DATASET2`, `COLON_ADENOMA`), as long as it is written identically in all six files. The notebook detects it from the UP and DOWN file names.

### Switching datasets

Only the two input file names at the beginning of the notebook need to change:

```python
up_file = DATA_DIR / "UP-50-GSE8671.xlsx"
down_file = DATA_DIR / "DOWN-50-GSE8671.xlsx"
```

For another dataset:

```python
up_file = DATA_DIR / "UP-50-DATASET2.xlsx"
down_file = DATA_DIR / "DOWN-50-DATASET2.xlsx"
```

The notebook then finds the matching GO and KEGG files using the same dataset name. The code logic does not need to change.

### Column requirements

The UP and DOWN gene-list files must contain a gene-symbol column. Recommended name:

```text
Gene.symbol
```

The GO/KEGG enrichment files must contain columns for the term or pathway name, the enriched genes and the adjusted p-value. Accepted column names include:

```text
Term, Description, Pathway, Name
Genes
Adjusted P-value, Adjusted_P_value, adj.P.Val, FDR, qvalue
```

## Workflow

1. Load the pre-filtered Top-50 up-regulated and Top-50 down-regulated gene lists.
2. Generate structured LLM-based biological interpretations (themes with direction, supporting genes, reasoning and confidence). No pathway names, statistics or enrichment results are given to the model in this step.
3. Run self-consistency analysis.
4. Apply API-based best-of-k candidate selection (three candidates).
5. Check for hallucinated genes: every gene cited by the model must be present in the input lists.
6. Compare the selected LLM themes with the real GO/KEGG enrichment results (theme level).
7. Generate LLM-predicted GO-like and KEGG-like profiles (ten terms per database and direction).
8. Compare the predicted profiles with the real Top-10 GO/KEGG terms (profile level) and compute Jaccard-type overlap indices.
9. Apply theme stability analysis.
10. Apply GO/KEGG significance caution flags.

## Comparison rule (Match / Partial / Unsupported)

Theme and reference-term names are lowercased and tokenized; tokens longer than two characters are kept, excluding stop words (*and, or, of, in, to, the, a, an, process, pathway, signaling, regulation, positive, negative, response*). Theme tokens are expanded with predefined synonym groups. For each reference term, the similarity score is the number of shared tokens plus twice the number of shared genes, and the highest-scoring term is retained. The theme or predicted term is then classified as:

| Category | Rule |
|---|---|
| **Match** | The best-scoring reference term shares **at least two genes** and **at least one token**, and has **adjusted p ≤ 0.10** |
| **Partial** | At least one shared gene or token, but not all Match criteria |
| **Unsupported** | No shared genes or tokens |

Theme-level evaluation compares each theme with all GO and KEGG terms for both directions. Profile-level evaluation compares each predicted term with the top ten enriched terms, ranked by adjusted p-value, in the same database and direction.

**Jaccard-type overlap index.** J = k / (20 − k), where k is the number of Match predictions (strict index) or Match + Partial predictions (partial index) among ten predictions compared with ten reference terms. Because one-to-one matching is not enforced, two predictions can be counted against the same reference term, so these values are count-derived upper bounds rather than a standard Jaccard set similarity.

**Significance labels:** adjusted p < 0.05 significant; 0.05–0.10 suggestive; > 0.10 not significant.

## Outputs

When a notebook is run, it writes the following files to the working directory. They are not stored separately in this repository; the results of the runs reported in the manuscript are shown in the PDF copies of the executed notebooks.

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

## Interpretation and limitations

* LLM outputs are **hypothesis-generating biological summaries**, not statistically validated enrichment results. The LLM does not calculate p-values, adjusted p-values, enrichment ratios or database-ranked statistics.
* A gene-level hallucination rate of 0% shows only that the cited genes were present in the input lists; it does not show that the functional claims are correct. Biological plausibility was therefore assessed separately (Supplementary Table S2).
* GO and KEGG served as a comparison reference, not as biological ground truth; lack of agreement does not imply that an LLM theme is wrong.
* The Partial category is permissive (one shared gene or token), so partial agreement indicates loose overlap rather than strong agreement.
* Results are based on two datasets, ten themes or terms per comparison, essentially one run per model and dataset, and one model family.

GO and KEGG remain necessary for database-grounded and statistically supported enrichment analysis.

## License, citation and disclaimer

**License.** See [`LICENSE`](LICENSE).

**Citation.** If you use this code or data, please cite the accompanying manuscript. The full citation will be added here after publication.

**Disclaimer.** This project is intended for academic and exploratory research. LLM-generated outputs are not clinical recommendations and must not be used as diagnostic or therapeutic evidence.
