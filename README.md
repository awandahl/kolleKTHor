

***

# kolleKTHor

**kolleKTHor** är ett kommandoradsverktyg som kompletterar DiVA-poster med saknade DOI:er genom att matcha mot Crossref. När en DOI har verifierats kan verktyget även (valfritt) hämta:

- Web of Science‑ID (ISI)
- Scopus EID
- PubMed‑ID (PMID)

Verktyget fungerar mot valfri DiVA-portal (t.ex. `kth`, `uu`, `umu`, `lnu`) och körs normalt på ett år i taget eller korta årsintervall.

## Huvudidé

1. Hämta publikationer från DiVA via export‑API:et för ett givet årsintervall.
2. Filtrera fram poster utan DOI (och ev. utan ISI/Scopus, beroende på inställningar).
3. Söka motsvarande poster i Crossref via titel + år.
4. Säkerställa träffar med:
    - publikationstyp (artikel, konferens, bok, kapitel)
    - volym, nummer, sidor
    - ISSN
    - efternamn på författare
5. För **verifierade** DOI:er:
    - Sätta kolumnen `Verified DOI`
    - Ev. föreslå `Possible DOI:s` om träffen är bra men inte fullt verifierad
    - Valfritt hämta ISI (WoS), Scopus EID, och PMID

***

## Funktioner

- Automatisk nedladdning av DiVA‑export som CSV
- Stöd för flera DiVA‑portaler via `DIVA_PORTAL`
- Matchning mot Crossref med justerbar titelsimilaritet
- Kontroll av:
    - volym, nummer, start-/slutsida
    - ISSN (journal/serie, tryckt/elektronisk)
    - minst ett överlappande författarefternamn
- Valfri berikning:
    - Web of Science‑ID (ISI) via DOI
    - Scopus‑ID (EID) via DOI
    - PubMed‑ID (PMID) via DOI
- Utdata i både CSV och Excel med klickbara länkar till:
    - DiVA‑post
    - DOI
    - Web of Science
    - Scopus

***

## Installation

1. Klona eller ladda ner projektet:
```bash
git clone https://github.com/<användare>/<repo>.git
cd <repo>
```

2. Installera beroenden (exempel):
```bash
pip install requests pandas tqdm xlsxwriter
```

Du behöver Python 3.9 eller senare.

***

## Konfiguration

All konfiguration ligger högst upp i skriptet (t.ex. `kolleKTHor.py`).

### Grundinställningar

```python
FROM_YEAR = 2025
TO_YEAR = 2025

DIVA_PORTAL = "kth"  # t.ex. "kth", "uu", "umu", "lnu"
MAILTO = "email@domain.com"  # Crossref kontaktadress
```

- `FROM_YEAR` / `TO_YEAR` anger vilket/vilka år som ska bearbetas.
- `DIVA_PORTAL` styr vilken DiVA‑instans som används (subdomänen innan `.diva-portal.org`).


### Val av vilka poster som ska bearbetas

```python
SCOPUS_ONLY = False
ISI_ONLY = False
BOTH_TYPES = False    # Scopus-only ELLER ISI-only (utan DOI)
NO_ID_ONLY = True     # poster utan DOI, utan ISI, utan Scopus
```

Exempel:

- Endast poster utan några identifierare:
`NO_ID_ONLY = True`, övriga `False`.
- Endast Scopus‑endast (utan DOI och ISI):
`SCOPUS_ONLY = True`, övriga `False`.
- Endast ISI‑endast:
`ISI_ONLY = True`, övriga `False`.
- Scopus‑eller‑ISI‑endast:
`BOTH_TYPES = True`, övriga `False`.


### Matchning mot Crossref

```python
SIM_THRESHOLD = 0.9           # minsta titelsimilaritet (0–1)
MAX_ACCEPTED = 9999           # max antal accepterade träffar innan scriptet stannar
CROSSREF_ROWS_PER_QUERY = 5   # hur många kandidater vi hämtar per titel
```

Högre `SIM_THRESHOLD` ger färre men säkrare träffar.

### Verifieringsflaggor

```python
VERIFY_USE_VOLUME = True
VERIFY_USE_ISSUE = True
VERIFY_USE_PAGES = True      # start+end som par
VERIFY_USE_ISSN = True       # kräver match på något ISSN
VERIFY_USE_AUTHORS = True    # kräver minst ett gemensamt efternamn
```

- Stäng av en parameter (sätt till `False`) om din DiVA‑data är svag på det området, t.ex. sidor.


### Proprietära databaser (WoS, Scopus)

Högst upp styr du om kommersiella API:er ska användas eller inte:

```python
USE_PROPRIETARY_WOS = True
USE_PROPRIETARY_SCOPUS = True
```

Dessa flaggor kopplas sedan till mer specifika inställningar:

```python
WOS_LOOKUP_FROM_VERIFIED_DOI = USE_PROPRIETARY_WOS
SCOPUS_LOOKUP_FROM_VERIFIED_DOI = USE_PROPRIETARY_SCOPUS
```

API‑nycklar:

```python
WOS_API_KEY = "*****"      # Clarivate Web of Science Starter API
SCOPUS_API_KEY = "*****"   # Elsevier / Scopus API
```

- Sätt `USE_PROPRIETARY_WOS = False` om du inte får/kan använda Web of Science.
- Sätt `USE_PROPRIETARY_SCOPUS = False` om du inte får/kan använda Scopus.
- När dessa är `False` görs inga anrop till respektive API, även om nycklarna finns kvar i filen.


### PubMed (alltid öppen)

```python
NCBI_TOOL = "kolleKTHor"
NCBI_EMAIL = "email@domain.com"
PUBMED_LOOKUP_FROM_VERIFIED_DOI = True
```

- `NCBI_TOOL` är ett namn för din klient (utan mellanslag).
- `NCBI_EMAIL` bör vara en giltig e‑postadress enligt NCBI:s rekommendationer.
- Om `PUBMED_LOOKUP_FROM_VERIFIED_DOI = True` försöker skriptet hämta PMID för varje verifierad DOI (om DiVA‑posten saknar PMID).


### Filnamn

Filerna namnges enligt:

```python
TIMESTAMP = datetime.now().strftime("%Y%m%d-%H%M%S")
PREFIX = f"{DIVA_PORTAL}_{FROM_YEAR}-{TO_YEAR}"

DOWNLOADED_CSV = f"{PREFIX}_diva_raw.csv"
OUTPUT_CSV = f"{PREFIX}_doi_candidates_{TIMESTAMP}.csv"
EXCEL_OUT = f"{PREFIX}_doi_candidates_links_{TIMESTAMP}.xlsx"
```

Exempel för KTH och år 2025 med tidsstämpel `20260224-111530`:

- `kth_2025-2025_diva_raw.csv`
- `kth_2025-2025_doi_candidates_20260224-111530.csv`
- `kth_2025-2025_doi_candidates_links_20260224-111530.xlsx`

***

## Hur skriptet arbetar (översikt)

1. Bygger en DiVA‑URL med årsfilter och publikationstyper.
2. Laddar ner CSV‑filen (`*_diva_raw.csv`).
3. Filtrerar rader enligt:
    - årsintervall
    - titel ej tom
    - typ (artiklar, kapitel, konferensbidrag, böcker, recensioner)
    - vald identifierarkombination (NO_ID_ONLY, SCOPUS_ONLY, ISI_ONLY, BOTH_TYPES).
4. För varje rad:
    - Hämtar upp till `CROSSREF_ROWS_PER_QUERY` kandidater från Crossref via titel + år.
    - Beräknar titelsimilaritet (tokenbaserad).
    - Kollar publikationstyp (artikel/konferens/bok/kapitel).
    - För kandidat(er) över tröskeln:
        - Hämtar full metadata från Crossref.
        - Jämför volym, nummer, sidor.
        - Jämför ISSN.
        - Jämför efternamn (minst ett gemensamt).
5. Om en kandidat klarar alla aktiva kontroller:
    - Markeras som **Verified DOI**.
    - Andra bra kandidater kan hamna i `Possible DOI:s`.
6. För varje **Verified DOI**:
    - Om WoS‑flagga är på: försök hämta ISI och fyll i `ISI` (endast numerisk del, utan `WOS:`).
    - Om Scopus‑flagga är på: försök hämta `ScopusId` (EID).
    - Om PubMed‑flaggan är på: försök hämta `PMID`.

***

## Utdata

Efter körning skapas:

1. **CSV med kandidater**

`*_doi_candidates_<TIMESTAMP>.csv` innehåller alla rader där antingen:
    - `Verified DOI` inte är tom, eller
    - `Possible DOI:s` inte är tom.

Kolumner inkluderar bl.a.:
    - `PID`, `DOI`, `Verified DOI`, `Possible DOI:s`
    - `ISI`, `ScopusId`
    - `Title`, `Year`, `PublicationType`
    - `Journal`, `Volume`, `Issue`, `StartPage`, `EndPage`, `JournalISSN`, `SeriesISSN`
    - `PMID`
    - `Name` (författare)
2. **Excel med klickbara länkar**

`*_doi_candidates_links_<TIMESTAMP>.xlsx` innehåller samma rader som CSV plus länkkolumner:
    - `PID_link` – DiVA‑posten
    - `Possible_DOI_link` – länk till DOI
    - `Verified_DOI_link` – länk till DOI
    - `ISI_link` – länk till Web of Science (om ISI finns)
    - `Scopus_link` – länk till Scopus (om ScopusId finns)

Länkarna skrivs som klickbara celler (t.ex. texten “PID”, “Verified DOI”) i Excel-arket.

***

## Exempel på körning

1. Öppna `kolleKTHor.py` och ställ in:
    - `DIVA_PORTAL = "kth"`
    - `FROM_YEAR = 2025`, `TO_YEAR = 2025`
    - `NO_ID_ONLY = True` (t.ex. bara poster utan DOI/ISI/Scopus)
    - `USE_PROPRIETARY_WOS = False` och `USE_PROPRIETARY_SCOPUS = False`
(för en “öppen” körning utan kommersiella API:er)
2. Kör skriptet:
```bash
python kolleKTHor.py
```

3. Granska resultat:
    - Öppna CSV‑filen i valfri editor för att se verifierade och möjliga DOI:er.
    - Öppna Excel‑filen för att klicka på länkar:
        - DiVA‑post (för manuell kontroll)
        - DOI‑länkar (Crossref/förlag)
        - Ev. WoS/Scopus (om proprietära flaggor är på och API:erna används)
    - Uppdatera sedan DiVA (manuellt eller via annat importflöde) med de DOI, ISI, ScopusId och PMID du bedömer som korrekta.

***

## Tips och begränsningar

- Titelsimilaritet och metadata kan vara bristfälliga, särskilt för äldre publikationer.
- Författarmatchningen använder endast efternamn, vilket kan ge både falska positiva och falska negativa i vissa fall.
- WoS- och Scopus‑API:erna kräver giltiga licenser och API‑nycklar.
- NCBI (PubMed) rekommenderar att du använder en stabil e‑postadress i `NCBI_EMAIL` och inte överbelastar deras API (skriptet väntar redan lite mellan anropen).

***

