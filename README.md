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

1. Harmonize the REDCap phenotype checklist export with a curated checklist-to-Human Phenotype Ontology (HPO) mapping.
2. Build the study cohort from the Gateway Kids Genome Project (GKGP, formerly the Undiagnosed Mendelian Disorders, or UMD, project) at WashU Medicine, keeping one clinical genetics note per participant (the note filed closest to, but not after, the date of manual chart review), and reshape the research coordinators' REDCap entries into a manual score table.
3. Score each note for every checklist phenotype with three large language models (LLMs): **GPT 5.4**, **GPT 5.4 Mini**, and **Claude Sonnet 4.6**.
4. Combine the manual and LLM scores, flag every participant-phenotype pair where any of the four scoring methods disagreed, and split those pairs into balanced sets for three expert adjudicators.
5. Correct the manual-score mapping, apply the final cohort exclusions, build a disagreement-adjudicated reference standard, and compare all four methods against the manual score (**Benchmark 1**) and the reference standard (**Benchmark 2**).
6. Fit Bayesian mixed-effects logistic regression models of factors associated with disagreement.
7. Generate the figures and tables in the manuscript and Supplementary Material.

The final analytic cohort is 637 participants, each contributing one note, scored on 124 phenotypes (78,988 participant-phenotype pairs per scoring method). Full methodological detail is in the manuscript and its Supplementary Methods.

## Important: no data are included

**This repository contains code only.** The underlying data are clinical notes and research records that contain protected health information (PHI) and cannot be shared publicly. As stated in the manuscript, the data will be shared on reasonable request to the corresponding author.

To prepare this code for public release:

- All notebook cell outputs have been cleared, so no tables, note text, or printed values from the study data appear in the notebooks.
- Hardcoded participant identifiers have been replaced with placeholders such as `[9 WES IDs]`. Cells containing these placeholders will not run as written.
- User-specific file paths have been replaced with the generic prefix `/Workspace/Users/user/user/`.
- API keys and model endpoints are read from Databricks secret scopes and are not stored in the code.
- Exploratory and troubleshooting cells that did not contribute to the reported results have been removed.
- A few markdown cells in the model notebooks record token counts and run times from the original runs. These contain no study data and are kept because they support Supplementary Tables 1 and 2.

Because the inputs are not available, these notebooks document how the analysis was performed rather than being runnable end to end. They can be adapted to other cohorts by supplying inputs with the same structure (see [Input files](#input-files)).

## Repository contents

| Notebook | Purpose |
| --- | --- |
| `GitHub_UMD_Data_Labels_Harmonization.ipynb` | Harmonizes the REDCap export (with data labels) against the checklist-to-HPO mapping file. Drops checklist items marked "ignore", resolves mismatched column names, and adds four brain imaging fields with their HPO terms. Writes the curated mapping (`hposetta_stone2.csv`) and harmonized REDCap labels (`data_labels_filtered4.csv`). |
| `GitHub_UMD_PreAI_Filtering.ipynb` | Builds the 668-participant cohort from the enrollment log, clinical notes, and REDCap records; reshapes the REDCap checklist into the manual score table; and prepares the filtered checklist used in the prompt. Writes `notes_enrolled4.csv`, `manual_score.csv`, and `checklist_filtered.csv`. |
| `GitHub_UMD_HPO_BaseGPT_Full_Run.ipynb` | Runs **GPT 5.4** on every note, including the within-budget pass, the oversized-note pass, and the targeted rerun, and merges them into a final output file. |
| `GitHub_UMD_HPO_Analysis_GPT_Mini_Full_Run.ipynb` | Runs **GPT 5.4 Mini** on every note and writes a final output file. |
| `GitHub_UMD_HPO_Sonnet_Full_Run.ipynb` | Runs **Claude Sonnet 4.6** on every note, including the successive reruns for incomplete or unparseable output, and merges them into a final output file. |
| `GitHub_AI_Outputs_to_Adjudication_Inputs.ipynb` | Combines manual and LLM scores into one participant-phenotype table, flags disagreements, rolls phenotypes up to depth-2 HPO categories, and allocates disagreements into a shared reliability set and three adjudicator sets. Also prepares the additional disagreements identified after the manual-score mapping correction. |
| `GitHub_Figures.ipynb` | Corrects the manual-score mapping, applies the final cohort exclusions, loads the completed adjudications, computes interrater reliability, builds the reference standard, and produces the figures and tables for both benchmarks. Saves the analysis tables used by the regression notebook. |
| `GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb` | Fits the Bayesian mixed-effects logistic regression models and draws the forest plots. Includes a data dictionary for its input tables. |

## Workflow

Run the notebooks in this order:

```
1. GitHub_UMD_Data_Labels_Harmonization.ipynb        (checklist-to-HPO mapping, harmonized REDCap labels)
        |
        v
2. GitHub_UMD_PreAI_Filtering.ipynb                  (cohort, manual scores, filtered checklist)
        |
        v
3. GitHub_UMD_HPO_BaseGPT_Full_Run.ipynb             (GPT 5.4)
   GitHub_UMD_HPO_Analysis_GPT_Mini_Full_Run.ipynb   (GPT 5.4 Mini)
   GitHub_UMD_HPO_Sonnet_Full_Run.ipynb              (Claude Sonnet 4.6)
        |
        v
4. GitHub_AI_Outputs_to_Adjudication_Inputs.ipynb    (disagreements -> adjudication files)
        |
        v
   [ Expert adjudication, performed in a separate interface not included here ]
        |
        v
5. GitHub_Figures.ipynb                              (benchmarks, reliability, figures, tables)
        |
        v
6. GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb
```

The three model notebooks in step 3 each read the outputs of step 2 and can be run independently of one another.

## Methods summary

### Phenotype checklist and HPO mapping

Implemented in `GitHub_UMD_Data_Labels_Harmonization.ipynb`. The REDCap checklist was linked to a curated mapping file that gives each checklist item its HPO term, HPO ID, and an include/ignore decision (HPO versions hpo-web@1.7.0 and hpo-obo@2020-08-11). Four brain imaging items without entries in the original mapping were added with their HPO terms. The full mapping is in Supplementary Table 8.

### Cohort construction

Implemented in `GitHub_UMD_PreAI_Filtering.ipynb` (see Supplementary Methods S3):

- Load the GKGP enrollment log, remove records missing enrollment date or date of birth, and collapse duplicate enrollments to the first enrollment event.
- For participants enrolled as minors, exclude notes filed after their 18th birthday.
- Keep notes authored by a physician, resident, nurse practitioner, fellow, or medical student.
- Join notes to the enrollment cohort and to the REDCap abstraction records, and drop participants with no REDCap data, no eligible note, or no date of last chart review. Seven missing review dates were imputed when the adjacent REDCap records agreed.
- Keep a single note per participant: the note filed closest to, but not after, the date of last chart review. This gives 668 participants, all of whom were sent to the LLMs.
- Reshape the REDCap checklist entries into a long table of manually scored phenotypes (`manual_score.csv`) and save the filtered checklist used to build the prompt (`checklist_filtered.csv`).

After LLM scoring, 31 notes were excluded for inadequate phenotyping content, incorrect note type, or incomplete documentation. These exclusions, and the removal of the nonspecific checklist term "Stenosis", are applied in `GitHub_Figures.ipynb`, giving the final 637 participants and 124 phenotypes.

### LLM extraction

- Each note was sent to each model on its own, with no other notes, manual scores, or adjudication results.
- Zero-shot, at temperature 0, with no fine-tuning, retrieval-augmented generation, or external tools.
- The same prompt was used for all three models. It frames the task as structured chart review by a board-certified medical geneticist and gives rules for clinical thresholds, developmental milestones, normal variation, and dysmorphology. The full prompt is in each model notebook and in the Supplementary Material.
- For each checklist phenotype the model returns JSON with a binary `score`, a `justification`, and a verbatim supporting `quote` from the note.
- Incomplete or unparseable responses were rerun until each participant had a complete set of scores. For GPT 5.4, notes exceeding the initial tokens-per-minute budget were set aside and run in a separate pass after the limit was raised, followed by a targeted rerun of incomplete outputs. For Claude Sonnet 4.6, nine successive reruns were needed. All runs and the merge into a final output file are kept in the notebooks; run times and token counts are summarized in Supplementary Tables 1 and 2.

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

The allocation was chosen from 1,000 random candidates (seeds 0 to 999) to minimize the spread in mean disagreements per participant across groups. Adjudication itself took place in a separate interface (not included here) that showed adjudicators the selected note only.

This notebook also downloads `hp.json` from the OBO PURL and rolls each phenotype up to its depth-2 ancestor under `HP:0000118` (Phenotypic abnormality) for ontology-level summaries.

After the manual-score mapping was corrected (Supplementary Methods S7), the manual positives that the original mapping had dropped were checked against the adjudication files. The 20 resulting disagreements that had not been adjudicated were added to one adjudicator's set as a separate extras file and adjudicated separately.

### Reference standard and statistics

`GitHub_Figures.ipynb`:

- corrects the manual-score mapping, applies the final cohort exclusions, and removes "Stenosis";
- loads the completed adjudications and, for each pair in the shared reliability set, selects one of the three ratings with a balanced random assignment (NumPy, seed 42) so each pair has a single reference label;
- merges in the 20 separately adjudicated pairs and joins all adjudication results to the final 637-participant cohort by Subject ID and phenotype;
- combines the adjudicated pairs (n = 4,358) with the pairs on which all four methods agreed (n = 74,630) to form the disagreement-adjudicated reference standard.

It then computes:

- precision, recall, F1, and related metrics from pooled confusion matrices for each benchmark;
- three-way percent agreement and Fleiss's kappa on the shared reliability set, plus pairwise percent agreement and Cohen's kappa; and
- how often adjudication supported the LLM score versus the manual score, for each direction of disagreement.

### Regression models

`GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb` fits `statsmodels` `BinomialBayesMixedGLM` models by variational Bayes for three comparisons, each fitted separately within reference-score strata of 1 and 0:

| Comparison | Pairs | Reference score | Reference method |
| --- | --- | --- | --- |
| Benchmark 1 | 78,988 (full cohort) | Manual score | GPT 5.4 |
| Benchmark 2, adjudicated subset | 4,358 | Adjudicated score | Manual |
| Benchmark 2, full cohort | 78,988 | Reference standard score | Manual |

Fixed effects are scoring method, standardized log note length, standardized age at note, and standardized note filing time, with crossed random intercepts for participant and phenotype. The notebook reports adjusted odds ratios with 95% credible intervals, approximate p-values, and the AUC of the fixed-effects predictions. Models that do not converge are refit with adjusted optimizer settings and checked against a rescaled comparison fit.

## Where each manuscript figure and table is produced

| Manuscript item | Notebook |
| --- | --- |
| Figure 1 (study design) | Not generated by code |
| Figure 2, Table 1 (Benchmark 1) | `GitHub_Figures.ipynb` |
| Figure 3 (adjudication outcomes of LLM vs manual disagreements) | `GitHub_Figures.ipynb` |
| Figure 4, Table 2 (Benchmark 2) | `GitHub_Figures.ipynb` |
| Supplementary Figure 1 (data processing workflow) | Not generated by code; row counts are printed by the notebooks in steps 1 to 3 |
| Supplementary Tables 1 and 2 (cost and processing time) | Calculated from the token counts and run times recorded in the three model notebooks |
| Supplementary Figure 2, Supplementary Table 3 | `GitHub_Figures.ipynb` |
| Supplementary Figures 3 and 4 (age at note, note length) | `GitHub_Figures.ipynb` |
| Supplementary Figure 5, Supplementary Table 4 | `GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb` (Benchmark 1) |
| Supplementary Table 5 (interrater agreement) | `GitHub_Figures.ipynb` |
| Supplementary Figure 6 (adjudicated pairs by age at note) | `GitHub_Figures.ipynb` |
| Supplementary Figure 7, Supplementary Table 6 | `GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb` (Benchmark 2, adjudicated subset) |
| Supplementary Figure 8, Supplementary Table 7 | `GitHub_Mixed_Effects_Logistic_Regression_Models.ipynb` (Benchmark 2, full cohort) |
| Supplementary Table 8 (checklist-to-HPO mapping) | `checklist_filtered.csv`, from `GitHub_UMD_Data_Labels_Harmonization.ipynb` and `GitHub_UMD_PreAI_Filtering.ipynb` |

## Input files

None of these files are included. They are listed so the code can be followed or adapted.

| File (as named in the code) | Description |
| --- | --- |
| `UMD Enrollment Log_11.7.2025_for_Gabriel.xlsx` | GKGP enrollment log (enrollment date and status, date of birth, MRN, consenting staff). A copy named `UMD Enrollment Log_11.7.2025_for_Gabriel 2.xlsx` is read in the adjudication notebook. |
| `umdhpo_notes_202605141230.csv` | Clinical genetics notes extracted from Epic Clarity (note text, filing time, author, author role, cosigner, note type, MRN) |
| `UndiagnosedMendelian-EHRExtraction_DATA_2026-05-05_1558.csv` | REDCap export (raw values) |
| `UndiagnosedMendelian-EHRExtraction_DATA_LABELS_*.csv` | REDCap export with data labels, including the phenotype checklist and date of last chart review. The 2026-05-05 export is used for harmonization and the 2026-05-26 export for cohort construction. |
| `UMD Phenotype Terms v4_final.csv` | Original checklist-to-HPO mapping with include/ignore decisions (refined into `hposetta_stone2.csv`) |
| `UMD Data for HPO Phecode mapping.csv` | Earlier REDCap extract used only as a reference during harmonization |
| `manual_score_dropped_positives.csv` | Manual positives dropped by the original manual-score mapping, identified during the mapping correction (Supplementary Methods S7) and used to find the additional disagreements sent for adjudication |
| `*_Kappa_Adjudications_Scored.csv`, `*_Note_Adjudications_Scored.csv`, `Dustin_Note_Adjudications_Extras_Scored.csv` | Completed adjudication files returned by each adjudicator (shared reliability set, individual sets, and the additional disagreements) |
| `hp.json` | HPO ontology, downloaded automatically from `http://purl.obolibrary.org/obo/hp.json` if not already present |

### Intermediate files

These are written by one notebook and read by a later one.

| File | Written by | Read by |
| --- | --- | --- |
| `hposetta_stone2.csv`, `data_labels_filtered4.csv` | Harmonization | PreAI_Filtering, Figures |
| `notes_enrolled4.csv`, `manual_score.csv`, `checklist_filtered.csv` | PreAI_Filtering | Model notebooks, Adjudication, Figures |
| `UMD_HPO_GPT_Output_Results_full_final.json`, `UMD_HPO_GPT_Mini_Output_Results_full_final.json`, `UMD_HPO_Sonnet_Output_Results_full_final.json` | Model notebooks | Adjudication, Figures |
| `*_Adjudications_Unscored.csv`, `Dustin_Adjudications_Unscored_updated.csv`, `Dustin_Adjudications_Extras_Unscored.csv` | Adjudication | Adjudication interface, Figures |
| `combined_df_full_info.csv`, `combined_df_full_info_adjudications.csv`, `gold_standard_full_info.csv` | Figures | Regression |

## Environment and requirements

The analysis was run in Python on the WashU Medicine Databricks environment. Some notebooks use Databricks-specific features (`dbutils.secrets`, `display()`, and `%pip` magics) and will need small changes to run elsewhere.

Python packages used:

- `pandas`, `numpy`, `scipy`, `scikit-learn`
- `statsmodels`, `patsy`
- `matplotlib` (3.8.4, pinned in `GitHub_Figures.ipynb`), `matplotlib-venn`, `upsetplot`
- `openai`, `anthropic`, `tiktoken`
- `openpyxl`, `python-dateutil`, `requests`

API keys and endpoints are read from a Databricks secret scope, for example:

```python
api_key = dbutils.secrets.get(scope="my-agent-secrets", key="openai-api-key")
azure_endpoint = dbutils.secrets.get(scope="my-agent-secrets", key="openai-endpoint")
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
