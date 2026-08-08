# Publisher runtime findings

Tested 2026-08-08 with the locally built Manifest V3 Connector loaded in Chrome for Testing 150. The final `saveItems` RPC was stubbed, so translation ran normally without writing to a Zotero library.

## Test controls

- Publisher translator source was seeded from the current translator repository tip (`fbee3268`), not the Connector's older bundled snapshot.
- `translators.attachSupplementary=true` and `translators.supplementaryAsLink=false` were applied to the background and offscreen translation contexts.
- ACS and Cell bot checks were allowed to clear in visible Chrome before detection was evaluated.
- Translation messages were directed to the article's top frame so an advertising frame could not produce an early empty response.

## Nature

Live page: <https://www.nature.com/articles/s41586-026-10843-7>

Verified real failures:

- Nature Publishing Group is detected and translates the correct title, DOI, URL, and main PDF.
- The page exposes 21 direct supplementary files (3 PDF and 18 XLSX), but the translator emits none of them because its `#supplementary-information` selector is obsolete.
- A broad modern selector would also see 10 Extended Data figure-page links. These must be excluded by requiring a genuine download/direct-media link.
- XLSX is absent from the legacy MIME map.

Not a failure on this page:

- Article URL targeting, DOI discovery, basic metadata, and main-PDF discovery all work.

## Cell Press

Live page: <https://www.cell.com/heliyon/fulltext/S2405-8440(24)04671-1>

Verified real failures:

- Cell Press is detected and the page exposes the correct DOI, main PDF, and one direct SI PDF.
- Before SI parsing, the legacy translator fetches the main PDF as text. The live request returns HTTP 403 and aborts the entire translation.
- When only that obsolete PDF probe is stubbed, metadata and the main PDF are emitted, but the SI file is still absent because the translator only understands the old `#main_supp` structure.
- Modern pages repeat SI in main and collateral panels, so URL deduplication is required.

Not a failure on this page:

- Translator targeting, DOI discovery, and Embedded Metadata extraction work. The earlier claim that this is only an SI-selector bug was incomplete because the PDF probe currently prevents any save.

## ACS Publications

Live page: <https://pubs.acs.org/jacsat/article/148/28/29684/5206327/Structural-Engineering-of-Cyanine-Dyes-to-Access>

Verified real failures:

- The ACS Publications translator does not match the new Silverchair article URL.
- The final article URL contains no DOI, while `meta[name="citation_doi"]` correctly provides `10.1021/jacs.5c22031`.
- The old ACS SI selector finds zero entries; the page exposes one `a[data-doctype="dataSupplementDoc"]` PDF.
- Zotero does not become translator-less: the generic Silverchair translator is detected instead.
- Silverchair's current `/Citation/Download` RIS request returns HTTP 403 in the Connector, and it saves zero items. Therefore switching ACS to that endpoint is not a sufficient fix.
- The page already supplies a usable `citation_pdf_url`, so the new path should not synthesize the old PDF URL.

False or overstated recommendations:

- “The new ACS URL leaves Zotero with no translator” is false: Silverchair, Embedded Metadata, and DOI are detected. The publisher-specific ACS translator is the missing detector.
- “Use the new Silverchair RIS endpoint” is contradicted by the live Connector run because that request receives 403. The ACS update should use page metadata and retain RIS only as a legacy fallback.
- The ACS `getDoi()` URL parser is obsolete, but it is not currently reached on the new page because ACS is not selected in the first place.

## Overall conclusion

The universal Connector preference bridge is necessary and test-covered, but it cannot repair publisher DOM, URL, or network behavior. Nature needs a focused modern-SI parser; Cell needs both a safe PDF path and modern-SI parsing; ACS needs a new-site translator path based on page metadata rather than the blocked RIS request.

## Post-fix verification

The implemented ACS, Nature, and Cell Press patches were then seeded into the same Chrome-for-Testing harness and run against the live pages:

- ACS: preferred ACS detection, correct DOI, main PDF, and one SI PDF.
- Nature: correct DOI, main PDF, 3 supplementary PDFs, and 18 XLSX files; all 10 figure-page links excluded.
- Cell Press: correct DOI, main PDF, and one deduplicated SI PDF without the prior 403 abort.
- ACS and Nature also passed link mode with identical SI URLs and `snapshot=false`.

The full non-live Connector suite passed with 102 tests.

An authorized save to the user's My Library confirmed full Nature byte transfer: the citation, main PDF, and all 21 SI files were accepted by Zotero. The ACS citation was accepted, but ACS blocked both binary attachment requests and the resolver fallback failed. A browser-challenge bypass probe timed out, confirming that ACS attachment transport remains a separate unresolved failure after translation.
