[README.md](https://github.com/user-attachments/files/33494452/README.2.md)
# Exclusivity Assay for Primer Design

## Overview

This Python script automates an **in-silico exclusivity check for PCR primers and probes** during assay design. It submits candidate oligonucleotide sequences to NCBI BLAST, collects sufficiently similar matches, and checks whether forward/reverse primer hits (and, for TaqMan assays, probe hits) form possible amplicons within a user-defined size range.

The script supports two assay modes:

- **TaqMan:** evaluates forward-primer, reverse-primer and probe combinations.
- **End-point PCR:** evaluates forward/reverse primer combinations without requiring a probe.

The workflow also lets the user exclude selected organisms from BLAST searches and review additional species interactively when a search returns close to the maximum number of alignments requested.

> **Important:** this is an in-silico screening tool. BLAST matches and predicted amplicons do not prove experimental specificity or exclusivity. Review the output and experimentally validate assay performance.

## Features

- Accepts multiple primers and probes interactively.
- Expands supported IUPAC ambiguity codes into concrete sequences before BLAST searches.
- Queries NCBI Taxonomy to resolve organism names to Taxonomy IDs (taxids).
- Submits remote `blastn` searches to NCBI using either `core_nt` or `nt`.
- Allows organism exclusions and optional exclusion of environmental samples.
- Runs up to three oligonucleotide jobs concurrently.
- Displays a live terminal status table with BLAST job status, retries, alignments, hits and exclusions.
- Saves per-iteration BLAST hit tables and final amplicon and species-summary tables.
- Includes retry handling for BLAST requests and XML parsing.

## Requirements

- Python installed on your computer.
- Internet access to reach NCBI BLAST and Entrez services.
- The Python package **Biopython**.

Install the dependency with:

```bash
python -m pip install biopython
```

The other imported modules are part of the Python standard library.

## Configuration before running

In the script, configure the email used to identify requests to NCBI:

```python
Entrez.email = "your_email@example.com"
```

Replace the value with an email address you use. The current script has an email address hard-coded in this line, so update it before sharing the script or publishing the repository.

The main timing and retry settings are defined near the top of the script:

| Parameter | Default | Purpose |
|---|---:|---|
| `poll_interval_sec` | `60` seconds | Interval between checks of a submitted BLAST job |
| `max_retries_blast` | `5` | Retry/restart limit used by the BLAST workflow |
| `max_retries_parse` | `5` | Maximum parse-retry count configured by the script |
| `MAX_POLLS_BEFORE_RESTART` | `15` | Number of polls before a BLAST job is restarted |
| `HTTP_TIMEOUT` | `60` seconds | Timeout for an individual HTTP request |
| `TABLE_REFRESH_SEC` | `10` seconds | Refresh interval for the live status table |

These settings can be adjusted in the script. BLAST is a remote service, so execution time depends on NCBI response times, the number and length of input oligonucleotides, ambiguity expansion and the amount of returned data.

## Running on Windows with the launcher (`run_script_on_Windows.bat`)

The project includes a Windows batch file named `run_script_on_Windows.bat`. This launcher makes it easier to start the pipeline on Windows without typing the Python command manually. It must be placed in the **same folder** as `exclusivity_assay.py`.

The folder should contain at least:

```text
project_folder/
├── exclusivity_assay.py
└── run_script_on_Windows.bat
```

### What the Windows launcher does

When you run `run_script_on_Windows.bat`, it:

1. Changes the working directory to the folder where the `.bat` file is located.
2. Checks that `exclusivity_assay.py` exists in that folder.
3. Looks for Python, preferring the Windows Python launcher command `py -3` and falling back to `python`.
4. Checks whether Biopython can be imported in that Python environment.
5. Runs `exclusivity_assay.py` and displays its exit code when it finishes.
6. Keeps the command window open until you press a key, so you can read any messages or errors.

The launcher **does not install Python or Biopython, update packages, or modify your environment**. If Python or Biopython is missing, it displays an error message and tells you what to check.

### Windows requirements and setup

- Install Python 3 and make sure `py -3` or `python` works from Command Prompt.
- Install Biopython for the Python environment used by the launcher. For example, open Command Prompt or a terminal and run one of the following, depending on which command works on your computer:

```bat
py -3 -m pip install biopython
```

or

```bat
python -m pip install biopython
```

- Keep `run_script_on_Windows.bat` and `exclusivity_assay.py` in the same folder.
- Double-click `run_script_on_Windows.bat` in File Explorer to launch the pipeline. Alternatively, run it from Command Prompt.

The `.bat` file is a convenience launcher only; the pipeline still runs the Python script and requires an internet connection to access NCBI services.

## Run the script directly

You can also run the Python script directly from a terminal opened in the directory containing it:

```bash
python exclusivity_assay.py
```

On Windows, you can use `py -3 exclusivity_assay.py` if the `py` launcher is available. The script asks for the following information in sequence.

### 1. Enter primers and probes

For each oligonucleotide, enter a name and its sequence. Type `done` at the name prompt when all oligonucleotides have been entered.

Example:

```text
Primer/probe name: forward_F1
Sequence: ATGCTGACTGACCTGA
Primer/probe name: reverse_R1
Sequence: TCGATGGTACCTTGAC
Primer/probe name: probe_P1
Sequence: ACGTACGTTGCA
Primer/probe name: done
```

Use names beginning with the appropriate role so the script can classify hits correctly:

- Names beginning with `forward` are treated as forward primers.
- Names beginning with `reverse` are treated as reverse primers.
- Names beginning with `probe` are treated as probes.

Names are converted to lowercase. For **TaqMan**, provide forward primer(s), reverse primer(s) and probe(s). For **End_point**, provide forward and reverse primers; probes are not required.

### IUPAC ambiguity codes

The script expands these symbols into all possible concrete sequences before submitting BLAST requests:

| Code | Bases represented |
|---|---|
| `Y` | C or T |
| `R` | A or G |
| `W` | A or T |
| `S` | G or C |
| `K` | T or G |
| `M` | C or A |
| `D` | A, G or T |
| `V` | A, C or G |
| `H` | A, C or T |
| `B` | C, G or T |
| `N` | A, C, G or T |
| `I` | A, C, G or T in this implementation |

Each expanded sequence is assigned an ID such as `forward_f1.1`, `forward_f1.2`, and so on. Because the script generates every combination, oligonucleotides containing several ambiguity positions can expand into many sequences and substantially increase runtime.

### 2. Enter organisms to exclude

At the organism-exclusion prompt, enter one organism or Entrez search term per line. Type `done` to finish.

- A standard organism name is resolved through NCBI Taxonomy and converted to a term of the form `txid<TAXID>[Organism:exp]`.
- Terms that start with `txid` or contain square brackets are passed through as advanced Entrez terms.
- If a name cannot be resolved in NCBI Taxonomy, the script asks you to enter it again.

The script also adds exclusions for records with `predicted[Title]` and `uncultured[All Fields]` to the BLAST Entrez query. You will then be asked whether to exclude environmental samples (`YES` or `NO`).

If a BLAST search returns at least 4,950 alignments, the script presents up to five species with the most accessions in that iteration. You can enter a species name to exclude it from subsequent iterations for that oligonucleotide, or type `skip` to stop adding exclusions for that oligonucleotide. The interactive taxonomic-exclusion loop is limited to six added taxids per oligonucleotide.

### 3. Choose the BLAST database

Enter one of the database names accepted by the script:

```text
core_nt
```

or

```text
nt
```

The script submits remote `blastn` jobs to NCBI; it does not download or build a local BLAST database.

### 4. Choose the PCR type

Enter one of the following:

- `Taqman`
- `End_point`

The input is converted to lowercase internally, so the corresponding values used by the script are `taqman` and `end_point`.

### 5. Define the amplicon size range

Enter the minimum and maximum amplicon sizes as integers. Candidate combinations outside this range are discarded during the downstream filtering step.

### 6. Name the output folder

Enter a name for the output folder, for example:

```text
exclusivity_results
```

The script creates the folder and its `blast_tables` subfolder if they do not already exist.

## What the script evaluates

The XML parser calculates percent identity using the number of identical positions divided by the original oligonucleotide length. Alignments below **70% identity** are excluded from the collected hit list. This threshold is set in the script and is not requested as a user input.

For both assay modes, the current implementation combines hits from oligonucleotides whose names begin with `forward` or `reverse`, then considers pairs on opposite strands that fall within the requested amplicon-size range. **It does not explicitly require one hit to come from an input named `forward` and the other from an input named `reverse`.** Instead, it labels the hit with the lower start coordinate as `Forward` in the output and the other as `Reverse`.

For **TaqMan**, the script additionally requires a probe hit whose start and end coordinates lie between the two primer-hit coordinates. It does not add a separate probe-strand check in this filter.

The current amplicon-size calculation uses the absolute difference between the two primer-hit **start** coordinates plus one. These implementation details matter when interpreting candidate combinations; inspect the input oligo IDs, per-iteration BLAST tables and final coordinates before drawing conclusions about assay exclusivity.

## Output files

For an output folder named `exclusivity_results`, the main outputs are:

```text
exclusivity_results/
├── blast_tables/
│   ├── <oligo_id>_iter1_hits.tsv
│   ├── <oligo_id>_iter2_hits.tsv
│   └── ...
├── exclusivity_results
└── exclusivity_results_summary
```

The exact number of BLAST tables depends on the number of iterations and whether an iteration contains qualifying hits. Tables are written only when the corresponding iteration has hits to save.

### `blast_tables/`

Each TSV represents one BLAST iteration for one expanded oligonucleotide. The table contains:

- Oligonucleotide ID and sequence.
- The Entrez query used for that iteration.
- Accession and organism description.
- Percent identity, subject start/end coordinates and strand.
- Number of mismatches, mismatch positions and gap intervals.

### Final amplicon table

The file with the same name as the output folder contains the combinations that pass the selected PCR-mode and amplicon-size filters. Columns include accession, organism, forward/reverse/probe IDs, coordinates, strands, amplicon size, mismatches, mismatch positions and gap intervals. In End-point mode, the probe-related fields are represented by `-` where no probe is used.

### Species summary table

The `<output_name>_summary` file summarizes the filtered results by species, including the number of accessions and the range of predicted copies per accession.

**Note:** the current script writes the final tables without filename extensions. You can add `.tsv` extensions in the code if desired, as long as the output paths are updated consistently.

## Live status table

During execution, a terminal table is refreshed periodically. It reports information such as:

- Oligonucleotide ID and NCBI BLAST request ID (`RID`).
- Current state, number of status polls and restart count.
- BLAST and parsing attempts.
- Number of alignments and accumulated qualifying hits.
- Species added to the exclusion list and final status.

Because the script uses parallel jobs and an interactive species-exclusion prompt, keep the terminal open and respond when a prompt requests input.

## Notes and limitations

- A working internet connection is required throughout the run.
- NCBI BLAST jobs can take time; avoid launching large numbers of jobs repeatedly and follow NCBI's service usage guidance.
- Ambiguity expansion can produce a combinatorial increase in the number of BLAST jobs.
- Hit collection is based on the current 70% identity threshold; it does not implement a configurable identity, coverage or thermodynamic threshold.
- The current amplicon filter does not explicitly enforce one hit from an input `forward` oligonucleotide and one from an input `reverse` oligonucleotide. It assigns the output labels according to coordinate order. Review this behavior before relying on the final table.
- The TaqMan filter checks whether the probe coordinates fall between the primer-hit coordinates but does not separately enforce probe strand orientation.
- The script's exclusions affect the Entrez-filtered BLAST search. They should be reviewed carefully to ensure that the intended organisms are excluded.
- Review the per-iteration BLAST tables as well as the final amplicon table. A species absent from the final table is not, by itself, proof that the assay will be exclusive against that species.
- Validate candidate primers/probes with appropriate experimental controls before using them in an assay.

## Troubleshooting

**The script cannot connect to NCBI**

Check your internet connection and try again later. NCBI service response times can vary. Confirm that the email configured in `Entrez.email` is valid.

**No final amplicons are reported**

Check the primer/probe names, PCR mode, amplicon-size range, organism exclusions, database choice and per-iteration hit tables. The filters may have removed all observed combinations.

**The run is very slow**

Check how many concrete sequences were generated from ambiguity codes. Consider testing with fewer oligonucleotides or less-degenerate sequences first.

**An oligonucleotide is not included in the final analysis**

Confirm that its name begins with `forward`, `reverse` or `probe`, as appropriate. Other prefixes are not assigned to the downstream forward/reverse/probe groups.
