---
title: "Supplementary File S1 — Prompt templates, inference settings and evaluation rules"
subtitle: "Evaluation of Large Language Models for Functional Interpretation of Differentially Expressed Genes (GSE8671 and GSE50114)"
---

# S1.1 Overview

This file documents the exact prompts, model settings, retry rules and rule-based evaluation procedures used in the LLM analyses. All content was taken directly from the four executed notebooks (one per model–dataset run).

| Run | Model | Dataset | Input gene files | Run date |
|---|---|---|---|---|
| 1 | GPT-OSS-20B | GSE8671 (discovery) | `UP-50-GSE8671.xlsx`, `DOWN-50-GSE8671.xlsx` | 12 June 2026 |
| 2 | GPT-OSS-120B | GSE8671 (discovery) | `UP-50-GSE8671.xlsx`, `DOWN-50-GSE8671.xlsx` | 25 June 2026 |
| 3 | GPT-OSS-20B | GSE50114 (validation) | `UP-50-VALIDATION.xlsx`, `DOWN-50-VALIDATION.xlsx` | 12 June 2026 |
| 4 | GPT-OSS-120B | GSE50114 (validation) | `UP-50-VALIDATION.xlsx`, `DOWN-50-VALIDATION.xlsx` | 25 June 2026 |

The same notebook code was used for every run. Between runs, only two values were changed: the model identifier (`MODEL_NAME`) and the input file paths. The dataset label (`GSE8671` or `VALIDATION`) is read from the input file name and used to select the matching GO/KEGG enrichment files (`GO-up-<dataset>.txt`, `GO-down-<dataset>.txt`, `KEGG-up-<dataset>.txt`, `KEGG-down-<dataset>.txt`). In the file names, `VALIDATION` refers to GSE50114. Each list contained 50 genes, giving 100 input genes per run.

# S1.2 Model access and inference settings

| Parameter | GPT-OSS-20B | GPT-OSS-120B |
|---|---|---|
| Model identifier | `openai/gpt-oss-20b` | `openai/gpt-oss-120b` |
| Provider | NVIDIA-hosted API | NVIDIA-hosted API |
| Interface | OpenAI-compatible Chat Completions | same |
| Message roles | one user-role message per call; no system message | same |
| Temperature | 0 (set explicitly in every call) | 0 |
| Top-p | not specified (provider default) | not specified |
| Random seed | not specified | not specified |
| Model snapshot | not pinned (hosted model) | not pinned |

Both models were called at the endpoint https://integrate.api.nvidia.com/v1. The hosted models are no longer available from the provider, so outputs cannot be regenerated exactly. All calls used the helper function below. Its default of `max_tokens=800` was overridden in every stage (Table S1.2).

```python
def get_completion(prompt, temperature=0, max_tokens=800):
    response = client.chat.completions.create(
        model=MODEL_NAME,
        messages=[{"role": "user", "content": prompt}],
        temperature=temperature,
        max_tokens=max_tokens
    )
    content = response.choices[0].message.content
    if content is None or content.strip() == "":
        return None
    return content.strip()
```

**Table S1.2. Stage settings (identical in all four runs).**

| Stage | Max output tokens | Outputs and retry rule |
|---|---|---|
| Direct annotation | 3,000 | 1 call |
| Structured reasoning | 3,000 | 1 output; up to 3 attempts. An attempt is accepted when the response contains the header and at least 11 table rows (header plus 10 rows). |
| Self-consistency | 3,000 | 3 outputs (k = 3); up to 12 attempts per output (overall cap 36). An attempt is accepted when at least 7 numbered lines each contain at least 2 input genes. |
| Best-of-k candidates | 3,000 | 3 candidates (K = 3); up to 5 attempts per candidate. An attempt is accepted when it parses to a list of exactly 10 objects. |
| GO/KEGG-like profile prediction | 8,000 | 4 profiles (GO up, GO down, KEGG up, KEGG down); up to 5 attempts per profile. An attempt is accepted when it parses to exactly 10 objects. |
| LLM verification (prompt S1.3.3) | 1,800 | Defined in the notebooks, but no call to it appears in the archived runs (see S1.6). |

Retry limits are maximum permitted attempts, not the number of requests made. Because temperature was 0, repeated self-consistency outputs and best-of-k candidates are repeated generations from the same prompt, not independent samples.

# S1.3 Prompt templates

Each block below is the exact Python f-string assigned to `prompt` in the notebooks. The six templates are identical, character for character, in all four runs. Expressions in braces (for example `{", ".join(up_genes)}`) are filled in before the call. Double braces (`{{` and `}}`) are Python escapes; the model received single braces.

## S1.3.1 Structured reasoning

Notebook cell index 3 (zero-based).

```python
f"""
You are a biomedical genomics expert.

Given the following differentially expressed genes from colorectal adenoma:

Upregulated genes:
{", ".join(up_genes)}

Downregulated genes:
{", ".join(down_genes)}

Task:
Generate a structured biomedical interpretation of the gene lists.

STRICT OUTPUT FORMAT:
Return only a markdown table.
The first line must be:
| Pathway_or_Process | Direction | Supporting_Genes | Biological_Rationale | Confidence |

Do not write any text before the table.
Do not write any text after the table.
Do not explain your reasoning process.
Do not include phrases such as "We need to", "Let's propose", or "Potential processes".

Rules:
- Use only genes provided in the input lists.
- Do not add genes that were not provided.
- Do not claim statistical significance for pathways.
- Avoid broad claims unless supported by multiple genes.
- Confidence must be High, Medium, or Low.
- Return exactly 10 rows only.
"""
```

## S1.3.2 Self-consistency

Notebook cell index 4 (zero-based).

```python
f"""
You are a biomedical genomics expert.

Given these differentially expressed genes from colorectal adenoma:

Upregulated genes in adenoma:
{", ".join(up_genes)}

Downregulated genes in adenoma:
{", ".join(down_genes)}

Task:
Generate 8 to 10 biological themes represented by these genes.

Return ONLY numbered lines.
Do not write introduction.
Do not write explanation before or after the list.

Format:
1. Biological process name: GENE1, GENE2, GENE3
2. Biological process name: GENE1, GENE2, GENE3

Rules:
- Return at least 8 numbered lines.
- Use only genes from the provided gene lists.
- Do not invent genes.
- Each line must contain at least 2 supporting genes.
- Use specific biological names, not Pathway1 or Biological_Process_Name.
- Do not claim statistical significance.
"""
```

## S1.3.3 LLM verification

Notebook cell index 7 (zero-based).

```python
f"""
You are a strict biomedical verification assistant.

Input gene lists:

Upregulated genes:
{", ".join(up_genes)}

Downregulated genes:
{", ".join(down_genes)}

Proposed LLM annotation:
{annotation_output}

Task:
Verify each pathway/process in the proposed annotation.

Return ONLY a markdown table.
Do not write any text before or after the table.

The table must have exactly these columns:
Pathway_or_Process | Verification_Status | Problem_Found | Corrected_Comment

Verification_Status must be one of:
Supported, Partially Supported, Unsupported

Rules:
- Check whether the supporting genes are present in the input lists.
- Check whether the gene-pathway relationship is biologically plausible.
- Mark broad or weakly supported claims as Partially Supported.
- Mark unsupported or invented claims as Unsupported.
- Do not add new pathways.
"""
```

`{annotation_output}` is the annotation being checked.

## S1.3.4 Direct annotation

Notebook cell index 13 (zero-based).

```python
f"""
You are a biomedical genomics expert.

You will receive two gene lists from differential expression analysis comparing
normal colorectal mucosa and colorectal adenoma.

Upregulated genes in adenoma:
{", ".join(up_genes)}

Downregulated genes in adenoma:
{", ".join(down_genes)}

Task:
Identify the main biological pathways or processes represented by these genes.

Return ONLY a markdown table.
Do not write explanations before or after the table.
Do not write your reasoning process.

The table must have exactly these columns:
Pathway_or_Process | Direction | Supporting_Genes | Explanation | Confidence

Rules:
- Use only genes from the provided lists.
- Do not invent genes.
- Do not claim causality.
- Do not claim statistical significance for pathways.
- Keep each explanation one sentence only.
- Confidence must be High, Medium, or Low.
- Return exactly 10 rows only.
"""
```

## S1.3.5 Best-of-k candidate generation

Notebook cell index 19 (zero-based).

```python
f"""
You are a biomedical genomics expert.

Given these differentially expressed genes from colorectal adenoma:

Upregulated genes in adenoma:
{", ".join(up_genes)}

Downregulated genes in adenoma:
{", ".join(down_genes)}

Task:
Generate exactly 10 biological themes represented by these genes.

Return valid JSON only.
Do not write any text before or after the JSON.
Do not use markdown fences.

JSON format:
[
  {{
    "theme": "theme name",
    "direction": "Upregulated",
    "supporting_genes": ["GENE1", "GENE2", "GENE3"]
  }}
]

Rules:
- Return exactly 10 objects.
- Use only genes from the provided gene lists.
- Do not invent genes.
- Each theme must use 2 to 5 supporting genes only.
- Theme names must be specific and biologically informative.
- Direction must be one of: Upregulated, Downregulated, Mixed.
"""
```

## S1.3.6 GO/KEGG-like profile prediction

Notebook cell index 22 (zero-based).

```python
f"""
You are a biomedical genomics expert.

Gene direction:
{direction}

Gene list:
{", ".join(gene_list)}

Task:
Predict the Top 10 {reference_type} results that are most likely represented by this gene list.

{reference_instruction}

Return valid JSON only.
Do not write any text before or after the JSON.
Do not use markdown fences.

JSON format:
[
  {{
    "rank": 1,
    "predicted_term_or_pathway": "specific biological process or pathway name",
    "supporting_genes": ["GENE1", "GENE2", "GENE3"],
    "confidence": "High",
    "biological_rationale": "One short sentence explaining why these genes support this result."
  }}
]

Rules:
- Return exactly 10 objects.
- Use only genes from the provided gene list.
- Do not invent genes.
- Each result must use 2 to 5 supporting genes.
- Do not provide p-values or adjusted p-values.
- Do not claim statistical significance.
- Confidence must be High, Medium, or Low.
- Do not use generic names such as Pathway1, Theme1, Process1, or Term1.
"""
```

## S1.3.7 Text inserted for `{reference_instruction}`

In prompt S1.3.6, `{reference_type}` was set to `GO_Biological_Process` or `KEGG_Pathway`, and `{reference_instruction}` was replaced by the matching block below. `{direction}` was `Upregulated` or `Downregulated`, and `{gene_list}` was the 50-gene list for that direction. The example names show the model the expected naming style. Because they could also prime its predictions, the predictions should not be described as completely unprompted.

**When reference_type = GO_Biological_Process:**

```text
Predict GO Biological Process-like terms.
Use biological process names such as:
cell cycle progression, regulation of Wnt signaling, cell adhesion,
immune response, ion transport, metabolic process.
Do not use KEGG-style pathway names.
```

**When reference_type = KEGG_Pathway:**

```text
Predict KEGG pathway-like names.
Use pathway names such as:
Wnt signaling pathway, Cell cycle, Metabolic pathways,
Cytokine-cytokine receptor interaction, Tight junction.
Do not use broad GO-style descriptions if a pathway-style name is possible.
```

# S1.4 Rule-based evaluation procedures

## S1.4.1 Gene-level hallucination check

A supporting gene was counted as hallucinated if it did not appear among the 100 input genes. For each theme, the hallucination rate is the number of hallucinated genes divided by the number of genes the model cited, × 100. For a whole candidate, the rate is calculated over all genes it cited.

## S1.4.2 Best-of-k candidate selection

Each of the three candidates was scored, and the candidates were sorted in this order:

1. lowest hallucination rate;
2. fewest Unsupported themes;
3. most Match themes;
4. highest Partial-or-Match rate.

The first candidate after sorting was selected. **For this selection step only**, each theme was labelled by the fixed keyword rule below, not by comparison with the enrichment results. The rules are checked in order on the lower-cased theme name, and the first rule that applies decides the label:

| Order | Theme name contains | Label |
|---|---|---|
| 1 | "wnt", "β-catenin", "beta-catenin" | Partial |
| 2 | "adhesion", "emt", "junction", "ecm", "remodeling", "invasion" | Match |
| 3 | "cell cycle", "cell-cycle", "proliferation", "mitotic" | Match |
| 4 | "metabolic", "metabolism", "glycolysis", "fructose", "amino acid" | Partial |
| 5 | "barrier", "transport", "epithelial", "ion" (matched anywhere inside a word) | Partial |
| — | none of the above | Unsupported |

This rule set was identical in all four runs. The selected candidate was then evaluated against the actual enrichment results (S1.4.3), and that second evaluation is the one reported in the manuscript.

## S1.4.3 Categorical comparison with GO/KEGG enrichment (Match / Partial / Unsupported)

This procedure was used to evaluate the selected best-of-k candidate and the GO/KEGG-like profile predictions.

**Tokenisation.** Names were lower-cased and split into alphanumeric tokens. Tokens of two characters or fewer were removed, and so were these stop words: and, or, of, in, to, the, a, an, process, pathway, signaling, regulation, positive, negative, response.

**Synonym expansion (theme side only).** If the theme name contained a trigger word, the listed tokens were added:

| Trigger in theme name | Tokens added |
|---|---|
| "cell cycle", "mitotic", "proliferation" | cell, cycle, mitotic, mitosis, division, proliferation |
| "dna", "replication" | dna, replication, nucleotide, synthesis |
| "apoptosis", "survival" | apoptosis, apoptotic, survival, death |
| "immune", "inflammatory", "inflammation" | immune, inflammatory, inflammation, cytokine, leukocyte, chemokine |
| "extracellular matrix", "ecm", "adhesion", "invasion" | extracellular, matrix, adhesion, migration, invasion, collagen |
| "metabolic", "metabolism", "oxidative" | metabolic, metabolism, oxidative, redox, biosynthesis |
| "angiogenesis", "vascular" | angiogenesis, vascular, vessel, blood |
| "wnt" | wnt, catenin |

**Best reference term.** Each reference term was scored as (number of shared tokens) + 2 × (number of shared genes). The term with the highest score was kept; ties went to the first term encountered.

**Classification.**

- **Match:** the best term shares at least 2 genes and at least 1 token with the theme, and its adjusted p-value is ≤ 0.10.
- **Partial:** at least 1 shared gene or at least 1 shared token, but not all Match criteria are met.
- **Unsupported:** no shared genes and no shared tokens.

**Reference set.**

- Theme-level evaluation of the selected candidate: all terms in the four enrichment files combined (GO and KEGG, upregulated and downregulated).
- Profile-level evaluation: the 10 terms with the lowest adjusted p-values in the same database and direction as the prediction.

# S1.5 Recorded runtimes

**Table S1.5. Recorded wall-clock time (seconds) for selected LLM steps in each model–dataset run.**

| Model | Dataset | Direct annotation | Structured reasoning | Self-consistency |
|---|---|---|---|---|
| GPT-OSS-20B | GSE8671 | 6.50 | 6.89 | 15.70 |
| GPT-OSS-120B | GSE8671 | 37.90 | 58.59 | 169.07 |
| GPT-OSS-20B | GSE50114 | 9.88 | 17.43 | 37.36 |
| GPT-OSS-120B | GSE50114 | 16.92 | 18.00 | 22.88 |

Times include network latency and provider-side queuing, so they reflect server load at the time of execution rather than model speed alone. Total pipeline time and input/output token counts were not logged.

# S1.6 Items not recorded

- Top-p and random seed were not set.
- No model snapshot or version was pinned.
- Token usage was not logged; the response `usage` field was not stored.
- Total end-to-end time was not logged.
- The best-of-k candidates that were not selected were not saved.
- The LLM verification prompt (S1.3.3) is defined in the notebooks, but no call to it appears in the archived runs.
