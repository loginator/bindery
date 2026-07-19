---
name: dnb-metadata
description: Use when working on the DNB (Deutsche Nationalbibliothek) metadata provider — the SRU API, MARC21 record mapping, or the author-catalogue edition/volume deduplication. Covers the query indexes, the 245 $a/$b/$n/$p title quirks, why DNB records proliferate (no "work" abstraction), the primary-provider wiring, and a live-probe recipe so you don't have to re-derive the catalogue's shape by hand.
---

# DNB metadata provider

DNB is Bindery's German-catalogue metadata source (`internal/metadata/dnb/`). It is an **enricher by default**, promoted to **primary** when `metadata.primary_provider = "dnb"` (recommended for German/Austrian/Swiss libraries where OpenLibrary coverage is thin). The switch is read once at startup in `cmd/bindery/main.go`; changing the setting needs a restart. When DNB is primary, OpenLibrary becomes the enricher.

## The SRU API

- Endpoint: `https://services.dnb.de/sru/dnb` — **no API key**, public.
- Protocol: SRU 1.1, `recordSchema=MARC21-xml`. Covers DE/AT/CH publications since 1913.
- Query indexes we use (`client.go`):
  | Index | Meaning | Used by |
  |-------|---------|---------|
  | `per=<name>` | works by a person | `SearchAuthors` (20), `GetAuthorWorks` (50) |
  | `tit=<title>` | title search | `SearchBooks` (20) |
  | `num=<id>` | control-number lookup | `GetBook`, name lookup in `GetAuthorWorks` |

Live probe (indispensable — the catalogue's shape is not guessable). Be gentle; DNB rate-limits and resets the connection under load:

```bash
curl -s "https://services.dnb.de/sru/dnb?version=1.1&operation=searchRetrieve&query=num%3D<CONTROLNUM>&maximumRecords=1&recordSchema=MARC21-xml" \
 | python3 -c "import sys,re; xml=sys.stdin.read()
for tag in ('020','245','264','490','800'):
 for m in re.findall(r'<datafield tag=\"%s\"[^>]*>(.*?)</datafield>'%tag,xml,re.S):
  subs=re.findall(r'<subfield code=\"(.)\">(.*?)</subfield>',m,re.S)
  print(f'{tag}:',' '.join(f'\${c}={v.strip()}' for c,v in subs))"
```

Use `query=per=Patrick%20Rothfuss` to see an author's full record set. (Grep with lazy `.*?` fails in POSIX `grep -oE`; use Python.)

## MARC → models.Book mapping (`recordToBook`)

| MARC | Field |
|------|-------|
| `001` | control number → `ForeignID` = `dnb:<001>` |
| `245 $a`/`$b` | title (`cleanDNBTitle` strips ` / `, ` | `, genre/promo subtitles) |
| `520 $a` | Description |
| `041 $a` | Language (3-char, e.g. `ger`) |
| `264 $c` / `260 $c` | publication year → ReleaseDate |
| `100 $a`, else `700` (`$4=aut`/`Verfasser`, else first) | Author, `"Last, First"` |
| `020 $a` | ISBN(s) → Editions (repeats per format) |

## Findings / gotchas (the expensive-to-rediscover part)

- **No author-ID lookup.** DNB's SRU exposes no authority-record endpoint, so `GetAuthor` errors for a real `dnb:<num>` and returns `(nil,nil)` for the synthetic IDs `dnb:gnd:<id>` / `dnb:author:<slug>` that `recordToBook` mints when a bib record has no linkable author. Consequences:
  - An author's `ForeignID` is actually a **publication** control number (from `SearchAuthors` reading `100 $a` off bib records), not a stable author/GND id.
  - The create flow (`api.fetchAuthorForCreate`) falls through to a synthesised author; its `metadataProvider` is derived from the `dnb:` prefix via `models.AuthorProviderFromForeignID` (do **not** hardcode `openlibrary`).
  - `GetAuthorWorks(dnb:<num>)` does `num=<id>` → read `100 $a` name → `per=<name>`.
- **No "work" abstraction.** OpenLibrary groups editions into one Work; DNB returns **one record per edition, printing, and physical volume**. A single German book therefore lands as many near-duplicate rows unless collapsed. Title-based dedup alone fails because each record's 245 string differs.
- **Volume markers live in three places** — reconcile all: an embedded `245 $a` marker (`(1)`, `/Band 1`), or `245 $n` (`Teil 1.`, `Tag 2`). A trailing bare number is **not** a volume (`Fahrenheit 451`).
- **`Teil`/`Band` vs `Tag` semantics differ.** `Die Furcht des Weisen / Teil 1|2` = one book physically split into two volumes → keep as two entries. `Die Königsmörder-Chronik / Tag 1` = a *series* position where Tag 1 is a complete book → resolve to the individual title.
- **`245 $p` names the individual work under a collective/series title.** e.g. `$a=Die Königsmörder-Chronik $n=Tag 1 $p=Der Name des Windes`. Prefer `$p`; the splitting volume is the `$n` **after** `$p` (the pre-`$p` `$n` is the series position). This merges series-catalogued editions with the standalone edition — a link **no shared ISBN reveals** (they are different printings with different ISBNs).
- **Series names appear as phantom "books."** A complete-series product titled with the series name (e.g. the audiobook `Die Königsmörder-Chronik : vollständige Lesung`) is not one of the author's books. Detect series names from the `490 $a` / `800 $t` statements the individual books carry, and drop records whose own title matches.
- **Combined + split editions coexist.** A work can have a one-volume "combined" edition (vol 0, `$b=Roman`) *and* numbered volumes. When numbered volumes exist, drop the redundant vol-0 combined edition.

## Where the code lives

- `internal/metadata/dnb/client.go` — SRU client, `SearchAuthors`/`SearchBooks`/`GetAuthorWorks`/`GetBook`, `recordToBook`, `cleanDNBTitle`.
- `internal/metadata/dnb/works.go` — author-catalogue collapse: `collapseAuthorWorks`, `seriesPartTitle`, `collectSeriesTitles`, volume parsing. Tests in `works_test.go` reproduce the real Patrick Rothfuss catalogue.
- `internal/metadata/dnb/types.go` — MARC XML structs + `subfield`/`subfieldAll`/`dataFieldsByTag`.
- `cmd/bindery/main.go` — primary-provider selection.
- `internal/metadata/aggregator*.go` — provider routing; `providerNameForForeignID` maps the `dnb:` prefix.

Related history: issue #667 (synthetic-ID hangs), #1574 (primary-provider dedup + edition collapse).
