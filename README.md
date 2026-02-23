

# kolleKTHor

kolleKTHor is a command‑line tool for enriching DiVA publication records with missing DOIs by matching against Crossref. It is designed to work for any DiVA portal (KTH, UU, UmU, Lnu, etc.) and to be run in year‑based batches.

The tool uses title, year, publication type, bibliographic metadata (volume/issue/pages, ISSN) and, when available, overlapping author surnames to identify high‑confidence DOI matches.

## Features

- **DiVA integration**
    - Connects to any DiVA portal’s `/smash/export.jsf` endpoint.
    - Filters on publication year (`FROM_YEAR`–`TO_YEAR`).
    - Restricts to selected publication types via `publicationTypeCode` (article, chapter, conference paper, book, review, book review).
    - Exports a configurable set of fields, including `Name` for authors.
- **Flexible identifier selection**
    - Processes only records that match your chosen identifier pattern:
        - `NO_ID_ONLY = True`: no DOI, no ISI, no ScopusId.
        - `SCOPUS_ONLY = True`: ScopusId only.
        - `ISI_ONLY = True`: ISI only.
        - `BOTH_TYPES = True`: Scopus‑only or ISI‑only.
- **Title and publication‑type aware Crossref search**
    - Queries Crossref by title and publication year.
    - Uses token‑based Jaccard similarity with a configurable threshold (`SIM_THRESHOLD`, default 0.9).
    - Maps DiVA `PublicationType` and Crossref `type` into coarse categories (article, conference, book, chapter) and **rejects mismatches**.
- **Verification using bibliographic metadata and authors**
For each promising DOI candidate from Crossref, KonnecTHor optionally checks:
    - Volume (`VERIFY_USE_VOLUME`)
    - Issue (`VERIFY_USE_ISSUE`)
    - Start/end pages (`VERIFY_USE_PAGES`)
    - ISSNs (`VERIFY_USE_ISSN`)
    - Author surnames (`VERIFY_USE_AUTHORS`)

A candidate becomes a **Verified DOI** only if all enabled checks that have data pass:
    - ISSN and volume/issue/pages must match (where present).
    - If `VERIFY_USE_AUTHORS = True`, there must be at least one overlapping surname between DiVA and Crossref.
- **Author parsing from DiVA**
    - Reads the DiVA `Name` column, which contains authors in the form:
`Family, Given [local-id] (affiliations…);Next, Author [...]`
    - Extracts clean display names (`Family, Given`) for reference work.
    - Derives author surnames (family names) for matching against `author[].family` from Crossref.
- **Two DOI outcome levels**
    - `Verified DOI`: strong match (title, type, year, and all enabled verification checks).
    - `Possible DOI:s`: good title/type match, but one or more verification checks missing or not fully convincing.
- **Rich outputs with links**
For each year range, KonnecTHor produces:
    - `{FROM}-{TO}_diva_raw.csv` – raw DiVA export for that slice.
    - `{FROM}-{TO}_doi_candidates.csv` – all records with either a verified or possible DOI.
    - `{FROM}-{TO}_doi_candidates_links.xlsx` – same rows, with clickable URLs for:
        - DiVA record (PID)
        - Possible / Verified DOI
        - ISI (Web of Science)
        - ScopusId


## Installation

```bash
git clone https://github.com/your-org/konnecthor.git
cd konnecthor
pip install -r requirements.txt
```

Requirements (typical):

- Python 3.9+
- `requests`, `pandas`, `tqdm`, `xlsxwriter`


## Configuration

All configuration is in the script header:

```python
FROM_YEAR = 1999
TO_YEAR = 1999

DIVA_PORTAL = ""   # e.g. "", "uu", "umu", "lnu"

SCOPUS_ONLY = False
ISI_ONLY = False
BOTH_TYPES = False
NO_ID_ONLY = True

SIM_THRESHOLD = 0.9
MAX_ACCEPTED = 9999
CROSSREF_ROWS_PER_QUERY = 5
MAILTO = "your.email@example.org"

VERIFY_USE_VOLUME = True
VERIFY_USE_ISSUE = True
VERIFY_USE_PAGES = True
VERIFY_USE_ISSN = True
VERIFY_USE_AUTHORS = True
```

Change `DIVA_PORTAL` to target another DiVA library (e.g. `"uu"` for Uppsala). Adjust `FROM_YEAR`/`TO_YEAR` to define the publication year slice to process.

If you do not want to use authors as a verification criterion, set:

```python
VERIFY_USE_AUTHORS = False
```


## Usage

1. **Edit configuration**

Open the script and set:
    - `DIVA_PORTAL` to the desired portal code.
    - `FROM_YEAR` and `TO_YEAR` for the publication years of interest.
    - Identifier selection flags (`NO_ID_ONLY`, `SCOPUS_ONLY`, `ISI_ONLY`, `BOTH_TYPES`).
    - Verification toggles as needed.
2. **Run KonnecTHor**

```bash
python konnecthor.py
```

3. **Review output**
    - Inspect `{FROM}-{TO}_doi_candidates.csv` and the corresponding Excel file.
    - Use the link columns to quickly check DiVA, Crossref, Scopus, and Web of Science.
    - Decide which `Verified DOI` and `Possible DOI:s` to feed back into DiVA (manually or via an import process).

### Typical workflows

- **Backfill a historical period**
Run in slices (e.g. 1990–1994, 1995–1999) with `NO_ID_ONLY = True` to find DOIs for older records that lack any identifiers.
- **Clean up Scopus‑only / ISI‑only records**
Set `NO_ID_ONLY = False` and enable `SCOPUS_ONLY`, `ISI_ONLY`, or `BOTH_TYPES` to turn existing Scopus/ISI‑only records into DOI‑identified records.
- **Deploy for another DiVA institution**
    - Change `DIVA_PORTAL`.
    - Test on a small year range to confirm that publication types, year parsing, and identifier patterns look correct.
    - Adjust verification toggles if volume/issue/pages or ISSN data are less reliable in that portal.


## Limitations

- Relies on DiVA’s export API and Crossref’s metadata; incomplete or inconsistent data on either side may prevent verification.
- Author matching uses only surnames and assumes DiVA `Name` follows the `Family, Given` convention.
- Very old publications and non‑journal items may lack enough structured metadata for full verification.

***



