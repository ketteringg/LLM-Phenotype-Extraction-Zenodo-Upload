# LLM Phenotype Extraction from Clinical Genetics Notes

Analysis code for the manuscript:

> **Large Language Model Phenotype Extraction from Clinical Genetics Notes: A Dual-Benchmark Evaluation in Rare Genetic Disease**
>
> Gabriel Kettering, Jiye Yi, Burak Altintas, Angie Carreño-Martinez, Marwan Shinawi, James Rudloff, Dustin Baldridge
>
> Department of Pediatrics, Washington University School of Medicine, St. Louis, MO, USA
>
> Submitted to the *Journal of the American Medical Informatics Association* (JAMIA), 2026.

---

## Overview

This repository contains the Python notebooks used to:

1. Build the study cohort from the Gateway Kids Genome Project (GKGP, formerly the Undiagnosed Mendelian Disorders, or UMD, project) at WashU Medicine, keeping one clinical genetics note per participant (the note filed closest to, but not after, the date of manual chart review).
2. Map the REDCap phenotype checklist used by research coordinators to Human Phenotype Ontology (HPO) terms.
3. Score each note for every checklist phenotype with three large language models (LLMs): **GPT 5.4**, **GPT 5.4 Mini**, and **Claude Sonnet 4.6**.
4. Compare LLM scores with the original manual research coordinator scores (**Benchmark 1**).
5. Identify every participant-phenotype pair where any of the four scoring methods disagreed and split those pairs into balanced sets for three expert adjudicators.
6. Build a disagreement-adjudicated reference standard and compare all four methods against it (**Benchmark 2**).
7. Fit Bayesian mixed-effects logistic regression models of factors associated with disagreement.
8. Generate the figures and tables in the manuscript and Supplementary Material.

The final analytic cohort is 637 participants, each contributing one note, scored on 124 phenotypes (78,988 participant-phenotype pairs per scoring method). Full methodological detail is in the manuscript and its Supplementary Methods.

## Important: no data are included

**This repository contains code only.** The underlying data are clinical notes and research records that contain protected health information (PHI) and cannot be shared publicly. As stated in the manuscript, the data will be shared on reasonable request to the corresponding author.

To prepare this code for public release:

- All notebook cell outputs have been cleared, so no tables, note text, or printed values from the study data appear in the notebooks.
- Hardcoded participant identifiers have been replaced with placeholders such as `[9 WES IDs]`. Code containing these placeholders will not run as written.
- User-specific file paths have been replaced with the generic prefix `/Workspace/Users/user/user/`.
- API keys and endpoints are read from Databricks secret scopes and are not stored in the code.

Because the inputs are not available, these notebooks document how the analysis was performed rather than being runnable end to end. They can be adapted to other cohorts by supplying inputs with the same structure (see [Input files](#input-files)).

## Repository contents

| Notebook | Purpose |
| --- | --- |
| `GitHub_UMD_Data_Labels_Harmonization.ipynb` | Harmonizes the REDCap export (with data labels) against the checklist-to-HPO mapping file. Drops checklist items marked "ignore", resolves duplicate and unmatched columns, and adds four brain imaging fields. Writes the curated mapping (`hposetta_stone2.csv`) and harmonized REDCap labels (`data_labels_filtered4.csv`). |
| `GitHub_UMD_HPO_BaseGPT_Full_Run.ipynb` | Builds the 668-participant cohort, reshapes REDCap scores into the manual score table, builds the prompt, and runs **GPT 5.4** on every note, including the oversized-note pass and targeted rerun. |
| `GitHub_UMD_HPO_Analysis_GPT_Mini_Full_Run.ipynb` | Same cohort construction and prompt, run with **GPT 5.4 Mini**. |
| `GitHub_UMD_HPO_Sonnet_Full_Run.ipynb` | Same cohort construction and prompt, run with **Claude Sonnet 4.6**, including the successive reruns for incomplete or unparseable output. |
| `GitHub_AI_Outputs_to_Adjudication_Inputs.ipynb` | Combines manual and LLM scores into one participant-phenotype table, flags disagreements, produces exploratory figures, and allocates disagreements into a shared reliability set and three adjudicator sets. |
| `GitHub_Figures.ipynb` | Corrects the manual-score mapping, applies the final cohort exclusions, loads the completed adjudications, computes interrater reliability, builds the reference standard, and produces the figures and tables for both benchmarks. Saves the analysis tables used by the regression notebook. |
| `GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb` | Fits the Bayesian mixed-effects logistic regression models and draws the forest plots. Includes a data dictionary for its input tables. |

## Workflow

Run the notebooks in this order:

```
1. GitHub_UMD_Data_Labels_Harmonization.ipynb
        |
        v
2. GitHub_UMD_HPO_BaseGPT_Full_Run.ipynb             (cohort + manual scores + GPT 5.4)
   GitHub_UMD_HPO_Analysis_GPT_Mini_Full_Run.ipynb   (GPT 5.4 Mini)
   GitHub_UMD_HPO_Sonnet_Full_Run.ipynb              (Claude Sonnet 4.6)
        |
        v
3. GitHub_AI_Outputs_to_Adjudication_Inputs.ipynb    (disagreements -> adjudication files)
        |
        v
   [ Expert adjudication, performed in a separate interface not included here ]
        |
        v
4. GitHub_Figures.ipynb                               (benchmarks, reliability, figures, tables)
        |
        v
5. GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb
```

The three model notebooks share the same cohort-construction and prompt-building cells, so each can be run on its own once step 1 is complete.

## Methods summary

### Cohort construction

Implemented in the first sections of each `*_Full_Run` notebook (see Supplementary Methods S3):

- Load the GKGP enrollment log, remove records missing enrollment date or date of birth, and collapse duplicate enrollments to the first enrollment event.
- For participants enrolled as minors, exclude notes filed after their 18th birthday.
- Keep notes authored by a physician, resident, nurse practitioner, fellow, or medical student.
- Join notes to the enrollment cohort and to the REDCap abstraction records, and drop participants with no REDCap data, no eligible note, or no date of last chart review. Seven missing review dates were imputed when the adjacent REDCap records agreed.
- Keep a single note per participant: the note filed closest to, but not after, the date of last chart review. This gives 668 participants, all of whom were sent to the LLMs.

After LLM scoring, 31 notes were excluded for inadequate phenotyping content, incorrect note type, or incomplete documentation. These exclusions, and the removal of the nonspecific checklist term "Stenosis", are applied in the downstream notebooks, giving the final 637 participants and 124 phenotypes.

### Phenotype checklist and HPO mapping

The REDCap checklist was linked to a curated mapping file that gives each checklist item its HPO term, HPO ID, and an include/ignore decision (HPO versions hpo-web@1.7.0 and hpo-obo@2020-08-11). The full mapping is in Supplementary Table 8. For ontology-level summaries, `GitHub_Figures.ipynb` downloads `hp.json` from the OBO PURL and rolls each phenotype up to its depth-2 ancestor under `HP:0000118` (Phenotypic abnormality).

### LLM extraction

- Each note was sent to each model on its own, with no other notes, manual scores, or adjudication results.
- Zero-shot, at temperature 0, with no fine-tuning, retrieval-augmented generation, or external tools.
- The same prompt was used for all three models. It frames the task as structured chart review by a board-certified medical geneticist and gives rules for clinical thresholds, developmental milestones, normal variation, and dysmorphology. The full prompt is in the `*_Full_Run` notebooks and in the Supplementary Material.
- For each checklist phenotype the model returns JSON with a binary `score`, a `justification`, and a verbatim supporting `quote` from the note.
- Incomplete or unparseable responses were rerun until each participant had a complete set of scores. Rerun cells and the merging into a final output file are kept in the notebooks.

| Model | Model identifier | Interface |
| --- | --- | --- |
| GPT 5.4 | `gpt-5.4` (API version `2025-01-01-preview`) | Azure OpenAI chat completions via `openai.AzureOpenAI`, JSON-object output enforced |
| GPT 5.4 Mini | `gpt-5.4-mini-pact` (API version `2025-01-01-preview`) | Azure OpenAI chat completions via `openai.AzureOpenAI`, JSON-object output enforced |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | Streaming messages via `anthropic.AnthropicFoundry`, JSON requested in the prompt |

All three models were accessed through WashU Medicine endpoints approved for PHI under a Business Associate Agreement.

### Adjudication

`GitHub_AI_Outputs_to_Adjudication_Inputs.ipynb` flags every participant-phenotype pair where any of the four methods (manual, GPT 5.4, GPT 5.4 Mini, Claude Sonnet 4.6) disagreed. Participants are allocated, at the participant level, to:

- a shared reliability set of 50 participants scored independently by all three adjudicators, and
- three individual adjudicator sets.

The allocation was chosen from 1,000 random candidates to minimize the spread in mean disagreements per participant across groups. Adjudication itself took place in a separate interface (not included here) that showed adjudicators the selected note only.

In `GitHub_Figures.ipynb`, one of the three ratings for each shared pair is selected with a balanced random assignment (NumPy, seed 42) so each pair has a single reference label. An additional 20 disagreements found after correcting the manual-score mapping (Supplementary Methods S7) were adjudicated separately and merged in. Adjudication results are joined to the final 637-participant cohort by Subject ID and phenotype.

### Reference standard and statistics

`GitHub_Figures.ipynb` combines the adjudicated pairs (n = 4,358) with the pairs on which all four methods agreed (n = 74,630) to form the disagreement-adjudicated reference standard. It computes:

- precision, recall, specificity, F1, and accuracy from pooled confusion matrices;
- three-way percent agreement and Fleiss's kappa on the shared reliability set, plus pairwise percent agreement and Cohen's kappa; and
- how often adjudication supported the LLM score versus the manual score, for each direction of disagreement.

`GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb` fits `statsmodels` `BinomialBayesMixedGLM` models by variational Bayes, separately within reference-score strata of 1 and 0. Fixed effects are scoring method, standardized log note length, standardized age at note, and standardized note filing time, with crossed random intercepts for participant and phenotype. It reports adjusted odds ratios with 95% credible intervals, approximate p-values, and the AUC of the fixed-effects predictions. Models that do not converge are refit with adjusted optimizer settings and checked against a rescaled comparison fit.

## Input files

None of these files are included. They are listed so the code can be followed or adapted.

| File (as named in the code) | Description |
| --- | --- |
| `UMD Enrollment Log_11.7.2025_for_Gabriel.xlsx` | GKGP enrollment log (enrollment date and status, date of birth, MRN, consenting staff) |
| `umdhpo_notes_202605141230.csv` | Clinical genetics notes extracted from Epic Clarity (note text, filing time, author, author role, cosigner, note type, MRN) |
| `UndiagnosedMendelian-EHRExtraction_DATA_2026-05-05_1558.csv` | REDCap export (raw values), including date of last chart review |
| `UndiagnosedMendelian-EHRExtraction_DATA_LABELS_*.csv` | REDCap export with data labels, including the phenotype checklist |
| `UMD Phenotype Terms v4_final.csv` | Original checklist-to-HPO mapping with include/ignore decisions (refined into `hposetta_stone2.csv`) |
| `UMD Data for HPO Phecode mapping.csv` | Earlier REDCap extract used only as a reference during harmonization |
| `*_Adjudications_Scored.csv`, `*_Kappa_Adjudications_Scored.csv`, `*_Note_Adjudications_Scored.csv` | Completed adjudication files returned by each adjudicator |

## Environment and requirements

The analysis was run in Python on the WashU Medicine Databricks environment. Some notebooks use Databricks-specific features (`dbutils.secrets`, `display()`, and `%pip` magics) and will need small changes to run elsewhere.

Python packages used:

- `pandas`, `numpy`, `scipy`, `scikit-learn`
- `statsmodels`, `patsy`
- `matplotlib` (3.8.4), `matplotlib-venn`, `upsetplot`
- `openai`, `anthropic`, `tiktoken`
- `openpyxl`, `python-dateutil`, `requests`

API credentials are read from a Databricks secret scope, for example:

```python
api_key = dbutils.secrets.get(scope="my-agent-secrets", key="openai-api-key")
```

To run elsewhere, replace these calls with environment variables or your own secrets manager. **Only send clinical text to model endpoints approved by your institution for PHI.**

## Ethics

The study was approved by the WashU Medicine Institutional Review Board (protocol 201504012). All clinical data were processed within institutionally approved environments, and LLM inference used institution-provided endpoints approved for PHI under a Business Associate Agreement. Reporting followed the TRIPOD-LLM guideline.

## Citation

If you use this code, please cite the manuscript:

> Kettering G, Yi J, Altintas B, Carreño-Martinez A, Shinawi M, Rudloff J, Baldridge D. Large Language Model Phenotype Extraction from Clinical Genetics Notes: A Dual-Benchmark Evaluation in Rare Genetic Disease. Submitted to *J Am Med Inform Assoc*. 2026.

## Acknowledgements

We are grateful to the families participating in the Gateway Kids Genome Project. Computational and storage resources were provided by WashU Medicine Research Infrastructure Services (RIS). Claude and ChatGPT were used to help generate analysis code, which was manually reviewed before use. See the manuscript for full funding and acknowledgement details.

## License

This code is released under the MIT License. See [`LICENSE`](LICENSE).

## Contact

Corresponding author: Dustin Baldridge, MD, PhD, Washington University School of Medicine (dbaldri@wustl.edu)
