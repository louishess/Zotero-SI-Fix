# Zotero supplementary-information fix handoff

## Next objective

Extend automatic supplementary-information (SI) downloads to additional online publishers while preserving Zotero's existing full-text behavior.

The main article PDF is already expected to download whenever the user's normal browser session has access—through open access, an individual subscription, or a university/library login. Treat that as a baseline invariant. SI support should be additive and must not replace, suppress, or reroute a working main-PDF attachment unless the publisher has actually retired the endpoint.

An HTTP 403 in an isolated test browser does not prove that a main-PDF URL is broken. Recheck it in the user's normal Chrome profile, where publisher cookies and institutional access are available.

## Ready-to-use local builds

The patched Zotero Desktop client is already built on this machine. A rebuild is not required for live publisher testing unless Desktop code or bundled translators change.

- Desktop source: `/Users/louishess/Developer/zotero-si-desktop`
- Built application: `/Users/louishess/Developer/zotero-si-desktop/app/staging/Zotero.app`
- Desktop branch: `fix/supplementary-preference-bridge`
- Desktop revision: `acf089d99`
- Connector source: `/Users/louishess/Desktop/Zotero Modification/Zotero SI Fix`
- Unpacked Chrome build: `/Users/louishess/Desktop/Zotero Modification/Zotero SI Fix/build/manifestv3`
- Connector branch: `fix/supplementary-preference-bridge`
- Connector revision before this handoff update: `cc813bd`

Quit the release Zotero application before launching the patched Desktop client. Chrome should have the store-installed Connector disabled and the unpacked build above enabled.

Required Desktop preferences:

- `extensions.zotero.translators.attachSupplementary=true`
- `extensions.zotero.translators.supplementaryAsLink=false`

The running patched client has already been verified to return `translatorPrefsVersion: 1`, `attachSupplementary: true`, and `supplementaryAsLink: false` from `/connector/ping`. It also supplies the patched ACS, Nature, and Cell Press translators.

## Repositories and branches

- Connector integration: <https://github.com/louishess/Zotero-SI-Fix/tree/fix/supplementary-preference-bridge>
- Publisher translators: <https://github.com/louishess/translators/tree/fix/supplementary-attachments>
  - current tip: `45d4b951`
- Zotero Desktop preference handoff: <https://github.com/louishess/zotero/tree/fix/supplementary-preference-bridge>
  - current tip: `acf089d99`
- Connector-compatible Zotero translator pin: <https://github.com/louishess/zotero/tree/fix/connector-supplementary-translators>
  - current tip: `b722f5c39`

## Universal functionality now working

- Zotero Desktop exposes primitive `extensions.zotero.translators.*` preferences through `/connector/ping`.
- The Connector applies them as runtime-only `translators.*` overrides and refreshes both injected and Manifest V3 offscreen translation contexts.
- Connector-local values return when Desktop disconnects.
- With the preference disabled, publisher translations retain their ordinary citation and main-PDF behavior without SI.
- With it enabled, a publisher translator can append SI descriptors in download mode or link-only mode.
- Final normal-profile testing retrieved both the main article PDF and SI for representative Nature, Cell Press, and ACS articles.
- The non-live Connector regression suite passes: 103 tests, with four opt-in live diagnostics skipped by default.

## Publisher fixes, easiest to hardest

### 1. Nature — modern DOM extraction

Nature was the least difficult because the article page already exposed direct, downloadable `media.springernature.com` URLs.

What worked:

- Replaced the obsolete `#supplementary-information` structure with modern `data-test` selectors.
- Required a terminal `(download TYPE)` label and a direct `media.springernature.com/original/` URL.
- Excluded Extended Data figure-detail pages that share Nature's SI link selector.
- Added modern DOCX and XLSX MIME mappings.
- Preserved link-only mode and all legacy selector fallbacks.

Live result for `10.1038/s41586-026-10843-7`: main article PDF plus 21 SI files (3 PDF and 18 XLSX), with no `/figures/N` false positives.

General lesson: first look for stable semantic attributes and direct file URLs already present in server-rendered HTML. Filter by URL and download semantics, not by titles such as “Extended Data.”

### 2. Cell Press — modern markup, deduplication, and CDN delivery

Cell Press required more work because its legacy SI page and selectors were obsolete, the same SI appeared in two page regions, and `www.cell.com` attachment URLs triggered Cloudflare responses in isolated Chrome.

What worked:

- Parsed modern `.core-supplementary-material` entries from the main and collateral sections.
- Deduplicated identical attachments by normalized absolute URL.
- Built useful titles from `.core-description` with filename/text fallbacks.
- Added PDF, DOCX, and XLSX handling while leaving unknown or large formats link-only.
- Removed an obsolete primary-PDF text probe that could abort item completion.
- In download mode, derived public `ars.els-cdn.com` URLs from the article PII and `mmc*` filename while preserving link behavior.

Live result for `10.1016/j.cell.2026.05.012`: the normal browser profile retrieved the main article PDF and both SI PDFs.

General lesson: deduplicate before attaching, and separate “the translator emitted the right attachment” from “the browser can fetch its bytes.” Prefer an official public CDN already used by the publisher over broad bot-bypass behavior.

### 3. ACS — platform migration, external SI API, and narrow network handling

ACS was the hardest because the publisher migrated to Silverchair, invalidating detection routes, DOI extraction, issue selectors, citation/PDF assumptions, and SI selectors. ACS SI download links also resolve through Figshare and its generic download hostname responds differently to a normal Chrome user agent.

What worked:

- Added Silverchair article, article-abstract, article-split, and issue routes.
- Read DOI and other metadata from current page metadata while retaining legacy fallbacks.
- Preserved the established `/doi/pdf/<DOI>` main-PDF route because it works with the user's ACS/browser access.
- Queried the ACS Figshare records by `resource_doi` for stable SI download URLs.
- Sorted numbered SI records, deduplicated files, and retained PDF, XLSX, ZIP, and MP4 MIME information.
- Added a narrowly scoped temporary User-Agent rule only for `https://ndownloader.figshare.com/files/` XHR downloads. Do not broaden this rule to `pubs.acs.org`.
- Preserved page-link and link-only fallbacks if Figshare discovery is unavailable.

Live result for `10.1021/jacs.5c22031`: the normal browser profile retrieved the main article PDF through ACS and the SI PDF through Figshare. Additional live transfers covered mixed PDF, XLSX, ZIP, and MP4 SI.

General lesson: after a full publisher-platform migration, treat detection, metadata, primary PDF, SI discovery, and binary delivery as separate layers. Repair and test each layer independently.

## Workflow for adding another publisher

1. Choose one representative article whose main PDF and at least one SI file are accessible in the user's normal Chrome session. Prefer an open-access example first; add subscription or university-access examples afterward.
2. Save once with `attachSupplementary=false`. Confirm citation metadata and the main article PDF still work. Record the exact main-PDF URL and do not change it without evidence.
3. Enable `attachSupplementary=true` and verify the Desktop preference appears in Connector background and offscreen contexts. If every publisher lacks SI, debug the universal handoff before touching translator code.
4. Inspect the live article HTML for stable semantic selectors, direct file URLs, format labels, duplicate page regions, and supplemental APIs.
5. Update only the publisher translator if it can already describe downloadable files. Add Connector-level network handling only when descriptors are correct but byte transfer consistently fails.
6. Infer MIME types from an explicit format, filename/download attribute, or API metadata before falling back to URL extensions. Unknown formats should remain links unless binary handling is proven safe.
7. Deduplicate normalized absolute URLs. Test repeated sidebar/modal markup and exclude figure-detail, preview, or navigation links.
8. Test three modes: preference off, download mode, and `supplementaryAsLink=true`.
9. Run descriptor-only tests in isolated Chrome, then perform one bounded personal-library save in the normal browser profile. Verify actual child attachments and file types in Zotero.
10. Keep each publisher change in its own translator commit. Commit any genuinely universal Connector change separately with unit coverage.

Useful initial inventory:

```sh
rg -l "attachSupplementary|getHiddenPref.*attachSupplementary" \
  src/zotero/translators
```

For each candidate publisher, record:

- representative URL and DOI;
- access type: open, personal subscription, or institutional;
- translator ID and revision;
- expected main-PDF URL;
- expected SI count, formats, and titles;
- DOM/API discovery method;
- duplicate and false-positive rules;
- descriptor-test result;
- real byte-transfer result.

## Regression and live tests

From the Connector repository:

```sh
./build.sh -d
env HEADLESS=true npm test
```

Existing publisher diagnostics without library writes:

```sh
env LIVE_PUBLISHER_TESTS=true EXPECT_PUBLISHER_FIXES=true \
  npx mocha --grep 'Live publisher translator diagnostics (ACS|Nature|Cell Press)$' \
  --timeout 120000
```

Link-only mode:

```sh
env LIVE_PUBLISHER_TESTS=true EXPECT_PUBLISHER_FIXES=true \
  LIVE_SUPPLEMENTARY_AS_LINK=true \
  npx mocha --grep 'Live publisher translator diagnostics (ACS|Nature)$' \
  --timeout 120000
```

Translator syntax:

```sh
cd src/zotero/translators
node .bin/check-syntax.mjs 'ACS Publications.js'
node .bin/check-syntax.mjs 'Nature Publishing Group.js'
node .bin/check-syntax.mjs 'Cell Press.js'
```

When adding a publisher, prefer a saved HTML fixture or intercepted page for deterministic SI counts. Keep a smaller live test to detect publisher drift.

## Known limitations and cautions

- Isolated Chrome profiles may receive publisher HTTP 403 responses even when a main PDF succeeds in normal signed-in Chrome. Do not weaken or replace working main-PDF behavior based only on that result.
- Publisher access remains authoritative. The changes do not bypass paywalls; they use access the user already has through open access, subscriptions, or library authentication.
- Zotero Desktop must be rebuilt from a checkout path without spaces. The already-built application above is ready for current live tests.
- The ACS issue-page multiple-selection flow still needs broader live coverage.
- `LIVE_LIBRARY_TRANSFER=true` writes to the currently selected personal Zotero library. Obtain explicit authorization, use a bounded citation count, and never delete the user's test citations unless asked.
- Existing authorized test citations and attachments were intentionally left in My Library for the user to remove.
