# KonnecTHor



KonnecTHor is a command‑line tool that downloads DiVA records, finds missing DOIs via Crossref, and outputs candidate and verified DOIs with linkable identifiers. To use it for another DiVA library you mainly change the portal code and year range.

## Main features

- **Works with any DiVA portal**
You set `DIVA_PORTAL = "kth"` / `"uu"` / `"umu"` / `"lnu"` etc., and the script calls that portal’s `/smash/export.jsf` API to get a CSV of publications for a given year range and a fixed set of publication types (article, chapter, conference paper, book, review, book review).
- **Flexible year slices and file naming**
`FROM_YEAR` and `TO_YEAR` define which publication years to include.
Output files are prefixed with the range, e.g.:
    - `1900-1995_diva_raw.csv` – DiVA export.
    - `1900-1995_doi_candidates.csv` – records with proposed/verified DOIs.
    - `1900-1995_doi_candidates_links.xlsx` – same rows with clickable links.
- **Identifier‑based selection of rows to process**

After loading the CSV, the script builds masks:
    - `has_doi`, `has_isi`, `has_scopus` from the corresponding columns.
    - `no_id_mask` = no DOI, no ISI, no Scopus.
    - `scopus_only_mask`, `isi_only_mask`.

With your current settings:
    - `NO_ID_ONLY = True`
    - `SCOPUS_ONLY = False`, `ISI_ONLY = False`, `BOTH_TYPES = False`

it will only process **records that have no DOI, no ISI, and no ScopusId** (purely “unidentified” items).
- **High‑precision title matching**

For each working row, KonnecTHor:
    - Queries Crossref `/works` with the **title** and **publication year**.
    - Computes a **Jaccard similarity** on tokenized titles.
    - Only considers candidates with similarity ≥ `SIM_THRESHOLD` (0.9), i.e. almost identical titles.
- **Publication‑type consistency**
    - DiVA’s `PublicationType` (e.g. `article`, `conferencePaper`, `book`, `chapter`, `review`, `bookReview`) is mapped to a coarse category (article, conference, book, chapter).
    - Crossref’s `type` (e.g. `journal-article`, `proceedings-article`, `book`, `book-chapter`) is mapped to the same set.
    - Candidates whose category doesn’t match DiVA’s category are skipped.
This ensures e.g. journal articles are only matched to Crossref journal articles, chapters to book chapters, etc.
- **Verification using volume/issue/pages/ISSNs**

For each title‑match candidate, the script optionally “upgrades” it from **Possible DOI** to **Verified DOI** if the Crossref metadata agrees with DiVA on:
    - Volume (`VERIFY_USE_VOLUME`),
    - Issue (`VERIFY_USE_ISSUE`),
    - Start/End pages (`VERIFY_USE_PAGES`),
    - ISSNs (`VERIFY_USE_ISSN`).

It fetches full Crossref metadata for the DOI, extracts volume/issue/page range and ISSN set, and prints a per‑field comparison. A candidate is marked as **verified** only if all enabled checks that have data match.
- **Two DOI statuses**

For each DiVA record the script may fill:
    - `Verified DOI` – high‑confidence match (title + type + biblio/ISSN checks).
    - `Possible DOI:s` – good title/type match but verification did not fully pass (or no detailed metadata was available).
- **Rich outputs with links**

The final Excel file adds link columns:
    - `PID_link` – DiVA record page.
    - `Possible_DOI_link`, `Verified_DOI_link` – `https://doi.org/...`.
    - `ISI_link` – Web of Science full record.
    - `Scopus_link` – Scopus record.

These cells are proper Excel hyperlinks; clicking them opens the identifier targets in your browser.


## How to use KonnecTHor for another DiVA library

You only need to change a few configuration values and rerun.

### 1. Point to the other DiVA portal

Set:

```python
DIVA_PORTAL = "uu"   # for Uppsala
# or "umu", "lnu", "bth", etc.
```

The script automatically builds:

```python
DIVA_BASE = f"https://{DIVA_PORTAL}.diva-portal.org/smash/export.jsf"
```

and uses that to export publications from that local DiVA.

### 2. Choose year range and identifier strategy

Set the publication years you want to process:

```python
FROM_YEAR = 2000
TO_YEAR = 2009
```

Decide which records to target:

- Only items with **no DOI, no ISI, no Scopus** (current behavior):

```python
NO_ID_ONLY = True
SCOPUS_ONLY = False
ISI_ONLY = False
BOTH_TYPES = False
```

- Only Scopus‑only items:

```python
NO_ID_ONLY = False
SCOPUS_ONLY = True
ISI_ONLY = False
BOTH_TYPES = False
```

- Only ISI‑only items:

```python
NO_ID_ONLY = False
SCOPUS_ONLY = False
ISI_ONLY = True
BOTH_TYPES = False
```

- Scopus‑only **or** ISI‑only:

```python
NO_ID_ONLY = False
BOTH_TYPES = True
SCOPUS_ONLY = False
ISI_ONLY = False
```


Then run:

```bash
python3 konnecthor.py
```


### 3. Review and use the outputs

After each run you get, for that portal and year slice:

- `{FROM}-{TO}_diva_raw.csv` – the raw export (good for debugging fields).
- `{FROM}-{TO}_doi_candidates.csv` – rows where KonnecTHor found either a verified or possible DOI.
- `{FROM}-{TO}_doi_candidates_links.xlsx` – same rows with convenient hyperlinks.

Typical workflow for a new DiVA library:

1. Run a **test slice** (e.g. 1–2 years) and inspect the Excel file to see if:
    - Publication types line up correctly (e.g. `article`, `conferencePaper` etc. really mean the same as at KTH).
    - The verification checks (volume/issue/pages/ISSN) are not too strict for that portal’s data quality.
2. If needed, adjust:
    - `SIM_THRESHOLD` if titles are consistently slightly off (e.g. 0.85 instead of 0.9).
    - Verification toggles, e.g. disable `VERIFY_USE_PAGES` if page data is unreliable for that library.
    - `diva_pubtype_category` if the portal uses slightly different `PublicationType` labels.
3. Once satisfied, run KonnecTHor over larger year blocks for that portal.

Because it relies only on the standard DiVA export API and Crossref’s REST API, the same script works across all DiVA member libraries with minimal, configuration‑only changes.

