# Detailed Plan: Port Reading/Book Tracking from Airtable to Capacities

This plan turns `high-level-plan.md` into concrete, verified steps. I researched the current
Jekyll setup, the two CSV exports, and the **new Capacities API v1** (the old Beta API is being
retired Sept 1, 2026). Findings that materially shape the plan are called out inline, and there
are **Open Questions** at the bottom for you to answer before I start.

---

## 1. Current state (verified)

### 1.1 What the site actually renders
`_layouts/reading.haml` reads `reading/books.json` and renders three things per year:
- **Currently Reading** — uses `Book.Title`, `Book.Authors`, `Started On`.
- **By year (finished)** — uses `Read In Year`, `Book.Title`, `Book.Authors`, `Finished On`,
  `Rating`, and per-year stats aggregated from `Book.Genres` and `Book.Author Background`.
- **Did Not Finish by year** — uses `Book.Title`, `Book.Authors`, `Stopped Reading On`.

> **Important:** the template does **not** render book covers, ISBNs, page counts, series,
> reviews, "recommended by", or date-added. Those exist in Airtable but are invisible on the
> site today. They matter only for fidelity of the Capacities database, not for the page.

### 1.2 The data contract (`reading/books.json`)
The sync script produces exactly this shape (this is the contract the new script must preserve):
```json
{
  "currently_reading": [ { "Started On": "YYYY-MM-DD", "Read In Year": "Not Yet Finished",
                           "Id": "...", "Book": { "Title": "...", "Authors": "...",
                           "Genres": [...], "Author Background": [...], ... } } ],
  "by_year": [ [ 2026, { "readings": [ { "Finished On": "YYYY-MM-DD", "Rating": 4,
                          "Read In Year": "2026", "Book": {...} } ],
                         "stats": { "Author Background": {"by-men": 3, ...},
                                    "Genres": {"sci-fi": 5, ...} } } ], ... ],
  "did_not_finish_by_year": { "2026": [ { "Stopped Reading On": "YYYY-MM-DD", "Book": {...} } ] }
}
```
Key rules from `sync_reading.rb` + `reading.haml`:
- `by_year` is an **array of `[year(Integer), data]` pairs, sorted descending**. `year` must be an
  Integer so `yearly_goals[year.to_i]` and month grouping work.
- `did_not_finish_by_year` is an **object keyed by year-as-string**.
- Dates are **ISO `YYYY-MM-DD`** (the template does `Date.parse(...)` on them).
- `Rating` is an Integer 1–5 or absent/null (template renders "Not Rated" when falsy).
- `Genres` / `Author Background` are **arrays of slug strings** (e.g. `"by-men"`, `"sci-fi"`).
- "Currently reading" = `Started On` present AND `Finished On` blank AND `Stopped Reading On` blank.
- "Finished" = `Finished On` present. "DNF" = `Stopped Reading On` present.

### 1.3 The CSV exports
- `Books-Master List.csv` — **851 books**. Columns: `Title, Authors, Cover, Series, Series Number,
  Number of Pages, ISBN, ISBN13, Recommended By, Genres, Author Background, Readings, Date Added,
  Want To Reread, For Lianne+Me Bookclub`.
- `Readings-Master View.csv` — **660 readings**. Columns: `Id, Book, Started On, Finished On,
  Stopped Reading On, Rating, Review, Read In Year, Page Count`.

Verified facts about the data:
- **Genres** slugs: `fantasy, fitness, mystery, nonfiction, sci-fi, tech`.
- **Author Background** slugs: `by-lgbtq-author, by-men, by-non-binary-authors,
  by-people-of-color, by-women`. (Both are multi-value, comma-separated in the CSV.)
- **Ratings** are `1`–`5`.
- **Dates in the CSVs are `M/D/YYYY`** (e.g. `9/24/2026`), *not* ISO — must be normalized.
- **Cover** column format: `filename.jpg (https://v5.airtableusercontent.com/...)`. As of now
  the URLs **still resolve (HTTP 200, image/jpeg)** — but per your note they expire within hours,
  so if we import covers via API we must do it in one fresh run or you re-export first.
- Each reading references exactly **one** book (no multi-book readings).

Data-quality issues I found (these need handling, see §2.3):
1. **Reading→Book linkage is by display value, not ID.** The CSV export has no Airtable record
   IDs. `Readings.Book` holds the book *title*; `Books.Readings` holds a date-range string like
   `"Mon 02-03-2020 to Tue 02-11-2020"`. Matching must be by normalized title.
2. **5 duplicate book titles** (`Changes`, `Cold Iron`, `Fated`, `Guards! Guards!`,
   `Ninefox Gambit`). Title-only matching is ambiguous for these — need a tiebreaker
   (e.g. author, or the date-range in `Books.Readings`).
3. **32 reading rows** reference a book title containing a comma; Airtable wraps those in
   embedded quotes (`"God, Human, Animal, Machine: ..."`). Normalization must strip surrounding
   quotes and leading/trailing whitespace before matching. (Some titles also have a leading space.)
4. At least one reading has `Finished On` earlier than `Started On` and future-dated entries
   (today is 2026-09-07). I will flag anomalies but not silently "fix" them.

---

## 2. Capacities API — what's possible (verified against the live OpenAPI spec)

Source of truth: `https://api.capacities.io/openapi.json` (v1) — the deprecated Beta spec at the
same host is being retired. Auth: `Authorization: Bearer <CAPACITIES_API_KEY>` with scopes
`api:read` / `api:write`. Version header `X-Capacities-Api-Version` may be required.

### 2.1 Endpoints that exist
- `GET /spaces`, `GET /space`, `GET /space/structures`
- `GET /object`, `POST /object`, `PATCH /object`, `DELETE /object`
- `POST /object/url`, `POST /object/markdown`, `GET /object/markdown`, `PATCH /object/markdown`
- Media upload (multipart): `POST /object/media/upload` (init) → `PUT /object/media/upload/part`
  → `POST /object/media/upload/complete` (or `/abort`)
- `POST /objects/search`, `GET /objects/structure`, `GET /objects/tag`, `GET /objects/collection`
- Block ops: `POST /blocks/append`, `PATCH /blocks/block`, `DELETE /block`

### 2.2 Hard constraints (these drive the whole strategy)
1. **You cannot create or modify structures/properties via the API.** Structures (object types)
   and their property definitions must be created **by hand in the Capacities app first**. The API
   only reads structure metadata (`GET /space/structures`) and writes objects into existing types.
   → So "create object schemas in Capacities" is a **manual UI step** I'll give you exact specs for.
2. **Creation quota:** a per-user hard cap of **5,000 objects total** (auto-increases +100/month).
   Books (851) + Readings (660) = **1,511 objects**; adding cover images as objects would be
   **+851 ≈ 2,362 total**. Under 5,000, but a meaningful and permanent chunk of your quota.
3. **Per-endpoint rate limits are tight** (the docs' own example shows `limit=5` per window) and
   returned in a `RateLimit: limit=…, remaining=…, reset=…` header; a 429 means wait `reset`
   seconds. Any API import must throttle and honor these headers + exponential backoff.
4. **Capacities explicitly recommends the in-app bulk-import feature over the API for bulk
   imports.** This is the single biggest fork in the plan — see Open Question A.

### 2.3 Property type mapping (Airtable → Capacities property types)
Capacities property types available: `title, text, number, boolean, url, date, label, entity,
aliases, icon, richText`. Proposed mapping:

**Book** structure:
| Airtable field | Capacities type | Notes |
|---|---|---|
| Title | `title` | primary |
| Authors | `text` | kept as single string to match current `Book.Authors` |
| Genres | `label` (multiple) | label options = the 6 genre slugs |
| Author Background | `label` (multiple) | label options = the 5 background slugs |
| Number of Pages | `number` | |
| ISBN / ISBN13 | `text` | |
| Series / Series Number | `text` / `number` | |
| Recommended By | `text` | |
| Date Added | `date` | |
| Want To Reread / Bookclub | `boolean` | |
| Cover | media object + `entity`, OR `url`, OR skip | see Open Question B |
| Readings | `entity` (multiple → Reading) | the relation |

**Reading** structure:
| Airtable field | Capacities type | Notes |
|---|---|---|
| Id (title) | `title` | e.g. "Mon 02-03-2020 to Tue 02-11-2020" |
| Book | `entity` (→ Book) | the relation |
| Started On / Finished On / Stopped Reading On | `date` | |
| Rating | `number` (or `label` 1–5) | number keeps template logic simple |
| Review | `richText` or `text` | |
| Read In Year | derived, probably omit | can be recomputed from Finished On |

- **Labels** are written as `{ "type":"label", "label":[{"id":"<optionId>","name":"..."}] }`.
  I must first read the option IDs from `GET /space/structures` after you create the label set.
- **Relations** are written as `{ "type":"entity", "entity":[{"id":"<objectId>"}] }`.
- **Dates**: `{ "type":"date", "date":{ "dateResolution":"day", "start":"YYYY-MM-DD" } }`.

---

## 3. Proposed approach

### Phase 0 — Prep & schema creation (manual, in the Capacities app)
0.1 You create two structures in the app: **Book** and **Reading**, with the properties in §2.3,
including the two multi-select label sets (Genres, Author Background) with the exact slug values,
and the Book↔Reading relation (entity properties on both sides).
0.2 I run `GET /space/structures` to capture the structure IDs, property definition IDs, and label
option IDs into a mapping file I'll use during import. (Read-only; safe to run anytime.)

### Phase 1 — Build & validate a local normalized dataset (no writes)
1.1 A parsing script reads both CSVs, normalizes dates to ISO, splits multi-value fields, strips
comma-quoting/whitespace, and **joins Readings→Books** by normalized title with a tiebreaker for
the 5 duplicates. It emits an intermediate JSON and a **validation report** (unmatched rows,
duplicate-title resolutions, date anomalies, blank required fields).
1.2 You review the report; we fix any real data problems before writing anything to Capacities.

### Phase 2 — Import into Capacities
Two candidate mechanisms (pick in Open Question A):
- **2A. In-app bulk import** — Capacities' recommended path for bulk. I'd shape the CSVs into the
  format its importer expects; you run the import in the app. Trade-off: label/relation/cover
  handling via CSV import may be limited, so some linking might need an API cleanup pass.
- **2B. API import script** — full control. Order: create all Books → create all Readings with the
  Book relation set → (optionally) upload covers and link them. Must throttle to the rate limit,
  honor `RateLimit`/429, be **idempotent/resumable** (persist a title→objectId map so a re-run
  doesn't duplicate), and stay within the creation quota. Covers, if included, are uploaded fresh
  in the same run because the source URLs expire.
1.x Either way I'll verify counts and spot-check records after import.

### Phase 3 — New `sync_reading.rb` (Capacities-backed)
3.1 Rewrite the script to pull from Capacities instead of Airtable:
- `GET /objects/structure?id=<Reading>` (paginated via `cursor`) to list readings, then
  `GET /object?id=…` per reading to get full properties + the linked Book. **Note:**
  `GET /objects/structure` returns only `{id, structureId, title}`, so full field values require a
  per-object GET — ~660 reads, which must be throttled to the rate limit (slower than Airtable's
  bulk fetch; caching/backoff needed).
- Reconstruct the exact `books.json` contract from §1.2 (currently_reading / by_year / DNF, ISO
  dates, integer years, genre/background stats, ratings).
3.2 Keep the API key in `.env` (already present as `CAPACITIES_API_KEY`); never commit it. Remove
the Airtable dependencies (`airrecord`, Airtable env vars) from the script and `Gemfile` if desired.
3.3 Validate by running the script and diffing the regenerated `books.json` against the current one
(structurally — the set of finished/DNF/current books and per-year stats should match, modulo any
data fixes from Phase 1). Compare `http://localhost:4000/reading/` before/after visually.

### Phase 4 — Handoff
4.1 Update `publish-reading.sh` only if the command name/flow changes.
4.2 Document the new workflow (how to add a reading in Capacities, then run the sync). No git
commits, no pushes to master, no site deploys — you handle all of that yourself.

---

## 4. Guardrails (from your instructions)
- **No `git commit`, no push to master, no site deploy** at any point — you do these yourself.
- `CAPACITIES_API_KEY` stays in `.env` and out of version control (`.gitignore` already covers
  `.env`; I'll confirm before any script runs).
- Nothing destructive in Capacities without your say-so; import scripts will be dry-run-first and
  resumable.

---

## 5. Deliverables
1. Structure specs for you to create in the app (Phase 0).
2. `import/` scripts: CSV parse+validate, structure-metadata fetch, and the importer (2A prep
   and/or 2B), plus a resumable title→ID map.
3. A rewritten `sync_reading.rb` backed by Capacities.
4. A short README describing the new workflow.

---

## 6. Open Questions — please answer inline below each one

**A. Import mechanism — in-app bulk import vs. API script?**
Capacities officially recommends the in-app bulk importer for bulk loads, and the API has tight
rate limits + a 5,000-object lifetime creation quota. The API gives cleaner control over relations,
labels, and covers but is slower and burns quota. Which do you prefer? (Options: "in-app bulk
import", "API script", or "whichever you judge best after Phase 1 — recommend one".)
> _Your answer:_ in-app bulk import

**B. Book covers — import them into Capacities at all?**
The site never displays covers, so they're not needed for `/reading/`. Importing them costs ~851
extra objects (quota) + many throttled upload calls, and the source URLs expire quickly. Options:
(1) skip covers entirely, (2) store the (expiring) Airtable URL as a `url` property, (3) download
all covers now and upload them as media objects linked to each Book, (4) download covers and commit
them into the site repo's `images/` instead. Which?
> _Your answer:_ 3) download and upload as media objects linked to each Book

**C. How faithful should the Capacities schema be?**
Do you want *all* Airtable fields ported (Series, ISBN/ISBN13, Recommended By, Page Count, Review,
Want To Reread, Bookclub, Date Added), or only the fields the site/sync actually needs (Title,
Authors, Genres, Author Background; Started/Finished/Stopped dates, Rating)? Full fidelity is more
setup work in Phase 0 but preserves everything.
> _Your answer:_ All of them

**D. Are the current CSVs authoritative and complete?**
They contain future-dated 2026 entries and at least one Finished-before-Started row. Should I treat
the CSVs as the source of truth as-is (importing anomalies verbatim), or do you want to correct the
data in Airtable and re-export first? Also: are 851 books / 660 readings the full history?
> _Your answer:_ Yes, they are authoritative—I'll fix forward any data issues.

**E. Duplicate-title tiebreaker for the 5 ambiguous books.**
To link readings to the right book when titles collide, is matching on **title + author** sufficient,
or should I fall back to the date-range string in `Books.Readings`? (If title+author still collides
for any of the 5, I'll list them for you to resolve manually.)
> _Your answer:_ Title and author is fine, treat any that match on both of those as duplicates.

**F. Should I remove the Airtable code path entirely?**
OK to drop `airrecord` and the `AIRTABLE_API_KEY`/`BOOKS_BASE_ID` usage from `sync_reading.rb` and
the `Gemfile`, or do you want the old script preserved (e.g. renamed `sync_reading_airtable.rb`) as
a fallback until you're confident in the Capacities pipeline?
> _Your answer:_ No, leave the Airtable code in place for now.

**G. Rating representation in Capacities.**
Store `Rating` as a `number` (1–5, simplest for the template) or as a `label`/select? Number is my
recommendation unless you want star-style selects in the Capacities UI.
> _Your answer:_
