# Datasets Explorer

Environmental datasets deposited by research groups · DAIS, Ca’ Foscari University of Venice

Datasets Explorer is a lightweight web application for browsing the research datasets that DAIS research groups have deposited in external data repositories: Zenodo, PANGAEA, Figshare, Datarepository Unive, Dryad and Harvard Dataverse. It lets anyone find a dataset, see who produced it, cite it correctly and open it in its repository.

The application is part of the tools developed by DESC – Data-centric Environmental Studies Centre, and shares its layout, typography and colours with the DAIS Herbarium.

## Architecture

The application has no server or database of its own:

```
Google Sheet (sheet DAIS\_Dataset\_Master of the dataset census)
        │  read
        ▼
Google Apps Script  ──  Code.gs  ──  public, read-only JSON API with caching
        │  fetch GET
        ▼
GitHub Pages  ──  index.html  ──  interface (HTML, CSS, JavaScript, no libraries)
```

Only the first sheet of the census is ever read. Audit sheets, scan queues and staff lists are never exposed by the API.

## What is published

* **Selection:** only rows whose *Valutazione AI Pertinenza* is `PERTINENTE` are published. Rows with any other value never leave the spreadsheet.
* **Scientific-disciplinary sectors:** codes are replaced by their full official names (DM 639/2024). For example, `GEOS-02/B` becomes *Geologia stratigrafica e sedimentologia*.

## Features

**Search and filters** (scrollable left column)

* Free-text search across title, authors, keywords, abstract, DOI and catalogue ID. It is accent-insensitive and highlights matching terms; when a term appears only in the abstract, the matching passage is shown.
* Filters:

  * repository;
  * publication year, either by tapping or dragging on a bar chart stacked by repository, or by typing a range;
  * DAIS author;
  * scientific-disciplinary sector;
  * keyword;
  * recent additions only.
* Every filter shows how many datasets each option would return, given the other active filters.
* Sorting by search relevance, year or title.
* Clickable rankings of the most frequent keywords, DAIS authors and sectors in the current results.

**Results**

* One row per dataset with:

  * title and year;
  * full author list, with DAIS authors in bold;
  * repository and DOI;
  * main keywords.

**Dataset record**

* Link to the dataset in its repository.
* Formatted citation and BibTeX entry, each ready to copy.
* DAIS authors with ORCID iDs.
* Abstract.
* Keywords and scientific-disciplinary sectors, each clickable to filter.
* Current version and related DOIs.
* The arrow keys ← → move through the results.

**Sharing and export**

* The address bar keeps the current search, so a filtered view can be shared as a link.
* The current results can be exported as CSV or as a BibTeX bibliography.

**General**

* Italian and English interface.
* Automatic light and dark theme.
* Responsive layout for desktop and mobile.
* Keyboard shortcuts: `/` to search, `Esc` to close a record.

## Setup

1. **Backend.** Open the census spreadsheet and choose Extensions → Apps Script. Paste `Code.gs`. If the script is a standalone project, set `SPREADSHEET\_ID`.
2. **Deploy the API.** Choose Deploy → New deployment → Web app, with *Execute as: Me* and *Who has access: Anyone*. Copy the URL ending in `/exec`.
3. **Test.** Open `…/exec?action=ping` and `…/exec?action=getVersion` in a browser.
4. **Configure the page.** In `index.html`, set `const API\_URL = '…/exec';`.
5. **Publish.** Upload `index.html` to this repository and enable GitHub Pages: Settings → Pages → Deploy from a branch → `main` / `(root)`.
6. **Automatic updates.** Reload the spreadsheet and choose **Data Explorer → Attiva aggiornamento automatico**. From then on, every change to the sheet clears the cache and appears on the site at the next page load.

After any change to `Code.gs`, publish a new version of the existing deployment (Manage deployments → Edit → New version) so the URL stays the same.

### Recommended OAuth scopes

To limit the script's permissions, enable *Show appsscript.json manifest file* in Project Settings and declare:

```json
"oauthScopes": \[
  "https://www.googleapis.com/auth/spreadsheets.currentonly",
  "https://www.googleapis.com/auth/script.container.ui",
  "https://www.googleapis.com/auth/script.scriptapp"
]
```

If the script is a standalone project rather than bound to the sheet, make two changes:

* use `https://www.googleapis.com/auth/spreadsheets.readonly` instead of `spreadsheets.currentonly`;
* remove `script.container.ui`, because the custom menu exists only in bound scripts.

## Configuration

|Setting|File|Purpose|
|-|-|-|
|`SOLO\_PERTINENTI`|`Code.gs`|`true` publishes only rows marked `PERTINENTE`|
|`SSD\_NAMES`|`Code.gs`|Full names of scientific-disciplinary sectors; add any new code that appears in the sheet|
|`OMIT\_ABSTRACT`|`Code.gs`|`true` leaves abstracts out of the API response, for a lighter page|
|`CACHE\_TTL\_SECONDS`|`Code.gs`|Lifetime of the server-side cache (maximum 6 hours)|
|`COLUMNS`|`Code.gs`|Accepted header names for each field; columns are recognised by name, so they can be reordered|
|`API\_URL`|`index.html`|Address of the Apps Script web app|
|`STATIC\_FALLBACK\_URL`|`index.html`|Optional static copy of the catalogue (for example `datasets.json`), used when the API does not respond|

## How it works

**Backend (`Code.gs`)**

* Reads the census sheet and normalises each row:

  * authors, keywords and sectors become lists;
  * ORCID iDs become a name → iD map;
  * related DOIs are extracted from the version notes.
* Caches the catalogue in Apps Script CacheService, split into 90 KB chunks, and computes a version hash. A browser that already holds the current version receives a short `notModified` reply instead of the whole catalogue.
* Adds a **Data Explorer** menu to the spreadsheet with three commands:

  * republish immediately;
  * run a metadata quality check, which counts missing abstracts, keywords, sectors and years, duplicate DOIs, and rows excluded as not pertinent;
  * enable automatic updates.

**Frontend (`index.html`)**

* A single self-contained file with no build step and no JavaScript libraries. The DESC logo is embedded in the file.
* Keeps a copy of the catalogue in the browser (`localStorage`), so the page opens instantly and stays usable when the API is slow.

## API

|Request|Response|
|-|-|
|`?action=getData`|`{version, updated, count, excluded, records:\[…]}`|
|`?action=getData\&v=<version>`|`{notModified:true}` when the catalogue is unchanged|
|`?action=getVersion`|`{version, count, updated}`|
|`?action=ping`|`{ok:true}`|

Every request accepts `\&callback=name` for JSONP. A record looks like this:

```json
{
  "id": "DAIS-DS-0001",
  "title": "…",
  "year": 2026,
  "repo": "Zenodo",
  "doi": "10.5281/zenodo.19045831",
  "url": "https://doi.org/10.5281/zenodo.19045831",
  "cite": "Author, A.; Author, B. (2026). Title. \[Dataset]. Zenodo. https://doi.org/…",
  "dais": \["Surname Name"],
  "orcid": {"Surname Name": "0000-0000-0000-0000"},
  "ssd": \["Informatica"],
  "kw": \["paleoclimate", "ice core"],
  "versions": \["10.5281/zenodo.21700605"],
  "version": "1.1",
  "abstract": "…",
  "status": "Consolidato"
}
```

## Privacy

The page sets no cookies and uses no analytics. The browser stores only the chosen interface language and the cached copy of the public catalogue (`localStorage`). The only external resource loaded is Google Fonts (Barlow, Spectral, IBM Plex Mono).

## Licence

© 2026 DAIS – Ca’ Foscari University of Venice

This project is licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International licence (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/). See the `LICENSE` file for the full legal text.

The following are not covered by this licence:

* the DESC and Ca’ Foscari logos;
* the datasets listed in the catalogue, together with their titles and abstracts, which remain under the licences set by their authors in the respective repositories.

[!\[CC BY-NC-SA 4.0](https://img.shields.io/badge/License-CC%20BY--NC--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc-sa/4.0/)

