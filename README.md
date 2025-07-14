# Pedigree AI inference

This repository contains a Python-based cloud function that orchestrates an end-to-end pedigreetree inference. The function performs the following steps:

1. **Fetch IBD metrics** for given sample tubeId(s).
2. **Fetch metadata** (e.g.,name, age, gender, haplogroups) for all related samples.
3. **Merge** IBD metrics with metadata.
4. **Build** a unique, ordered family-tree CSV of close relatives.
5. **Subset** the family tree up to grandparents.
6. **Infer pedigrees** via ChatGPT-based inference.

The function is currently invoked via a simple CLI.

---

## Prerequisites

- **Python** 3.12 installed.
- Ensure the following Python packages are available (via `pip install`):
  - `python-dotenv`
  - `pandas`
  - `pymongo`
  - `mysql-connector-python`
  - `openai`

## Configuration

- The function loads environment variables from a production file path hardcoded as `env_path = '/path_to/prod.env'`. No additional configuration is required; just ensure that `prod.env` exists and contains the necessary MongoDB and MySQL credentials.


## CLI Invocation

Run the pipeline locally or within the cloud function via:

```bash
python main.py --tubeids <comma-separated-tubeid(s)>
```

**Example:**

```bash
python main.py --tubeids aa1111,bb2222,cc3333
```

This command will:

1. Create an output directory `fam_<suffix>/` (where `<suffix>` is a concatenation of the first five tubeIds).
2. Generate intermediate CSVs:
   - `metadata_<suffix>.csv`
   - `ibd_metrics_<suffix>.csv`
   - `merged_meta_ibd_<suffix>.csv`
   - `relatives_uniq_ordered_<suffix>.csv`
   - `subset_relatives_<suffix>.csv`
3. Invoke the pedigree inference step, producing a final JSON file `fam_<suffix>.json`.
4. Write detailed logs to `fam_<suffix>/fam_<suffix>.log`.

---

## Module Overview
      
- `Get_IBD_Metrics.py`: Recursively fetches and cleans half-IBD pair metrics from MySQL/MongoDB sources.
      Returns a dataframe and outputs it as csv:

    | id1    | id2    | ibd_sum | ibd_n | ibd_max |
    |--------|--------|---------|-------|---------|
    | aa1111 | ee5555 | 3602.82 | 79    | 242.506 |
    | cc3333 | bb2222 | 2063.54 | 24    | 229.148 |
    | cc3333 | dd4444 | 3545.64 | 23    | 283.677 |

- `Get_Metadata.py`: Retrieves sample metadata from MongoDB, computes age, reconciles gender, and merges haplogroups.
      Retruns a dataframe and outputs it as a csv:

    | full_name                | age | meta_gender | patient_id | tube_id | MT_hap_apex | Y_hap_apex |
    |--------------------------|-----|-------------|------------|---------|-------------|------------|
    | surname name middleName  | 34  | Male        | 11111      | aa1111  | T1A1N       | N-TAT      |
    | surname name middleName  | 53  | Female      | 22222      | bb2222  | T1A1N       |            |
    | surname name middleName  | 8   | Female      | 33333      | cc3333  | H5E1        |            |

- `Merge_Metadata_IBD.py`: Merges metadata (twice) into IBD metrics DataFrame and writes the merged CSV.
- `Get_FamilyTree.py`: Constructs an ordered, deduplicated family-tree table of relatives from genealogical cards and trees.
      Returns a dataframe and outputs it as a csv:

    | full_name                 | tubeId  | patientId | id     | spouse1    | parent1     | parent2      | child1     |
    |---------------------------|---------|-----------|--------|------------|-------------|--------------|------------|
    | surname name middleName   | aa1111  | 11111     | XxXxXx |            | XxXxXx      | XxXxXx       |            |
    | surname name middleName   | bb2222  | 22222     | XxXxXx | XxXxXx     | XxXxXx      | XxXxXx       | XxXxXx     |
    | surname name middleName   | cc3333  | 33333     | XxXxXx | XxXxXx     | XxXxXx      | XxXxXx       | XxXxXx     |


- `Subset_FamilyTree.py`: Filters the full family tree to include only up to grandparents, ensuring minimal dataset for inference.
      Returns and outputs the same as  `Get_FamilyTree.py`,
- `ChatGPT_Inferrence.py`: Reads merged and subset CSVs to infer pedigrees via GPT API.
- `main.py`: Orchestrates the full pipeline, handles arguments, logging, and output management.
      Outputs and logs a pedigree table (method: corrected from family tree, ibd-only):

    | id1    | id2    | ibd_sum | ibd_n | pedigree               | method    | reason                                                       |
    |--------|--------|---------|-------|------------------------|-----------|--------------------------------------------------------------|
    | aa1111 | ee5555 | 3602.82 | 79    | full_siblings          | corrected | Shared both parents (bb2222 & ff6666) in tree                |
    | cc3333 | bb2222 | 2063.54 | 24    | grandparent-grandchild | corrected | Tree shows bb2222 is grandfather of cc3333                   |
    | cc3333 | dd4444 | 3545.64 | 23    | parent-child           | corrected | Tree shows dd4444 is mother of cc3333                        |
    | cc3333 | ee5555 | 1909.75 | 47    | uncle-niece            | corrected | Tree shows ee5555 is cc3333’s paternal uncle                 |

    and a JSON dictionary:

    <pre> { "fam": [ { "family_id": "Network#", "ind_id": "aa1111", "father_id": "ff666", "mother_id": "bb222", "gender": 1, "phenotype": 0 }, ... ],
      "samples": ["aa1111", "bb2222", "cc3333", ...] } </pre>
---

## Output Structure

Upon successful execution, the output directory `fam_<suffix>/` will contain:

- **CSVs**:
  - `metadata_<suffix>.csv`
  - `ibd_metrics_<suffix>.csv`
  - `merged_meta_ibd_<suffix>.csv`
  - `relatives_uniq_ordered_<suffix>.csv`
  - `subset_relatives_<suffix>.csv`
- **Logs**:
  - `fam_<suffix>.log`: Comprehensive stdout and stderr.
- **Final JSON**:
  - `fam_<suffix>.json`: Extracted pedigree inference JSON.

---

## Logging & Troubleshooting

- All `print` and function messages are captured to both console and the `.log` file by `TeeStdout` in `main.py`.
- In case of any unhandled error, inspect `fam_<suffix>.log`.
- Common issues:
  - **Missing credentials**: Ensure `prod.env` is accessible.

---
