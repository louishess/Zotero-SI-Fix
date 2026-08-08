# Zotero supplementary-information fix handoff

## Repositories and branches

- Connector integration: <https://github.com/louishess/Zotero-SI-Fix/tree/fix/supplementary-preference-bridge>
  - tip: `c14fc9b6c0d80ea1f37b525b41874168de5a1a66`
- Publisher translators: <https://github.com/louishess/translators/tree/fix/supplementary-attachments>
  - tip: `0d77dbaf1541f31bf3dac73611fefa060053f777`
- Zotero Desktop preference handoff: <https://github.com/louishess/zotero/tree/fix/supplementary-preference-bridge>
  - tip: `1cec11b73303463809d8da263f123c42a957e5dd`
- Connector-compatible Zotero translator pin: <https://github.com/louishess/zotero/tree/fix/connector-supplementary-translators>
  - tip: `04d0ab8daaf170b4207a820207cde2461279eb6e`

## What works

- Zotero Desktop can expose namespaced translator preferences in `/connector/ping`.
- The Connector applies those values as runtime-only overrides, refreshes injected and Manifest V3 offscreen translation contexts, and restores Connector-local values after disconnect.
- Connector builds succeed when the checkout path contains spaces.
- Nature's modern SI parser emits 3 PDF and 18 XLSX files for `s41586-026-10843-7` and excludes all 10 Extended Data figure-page links.
- Nature personal-library transfer succeeded: citation, main PDF, and all 21 SI files were accepted by Zotero.
- ACS's new Silverchair URL is detected by the ACS translator. SI discovery resolves ACS Figshare records and transfers PDF, XLSX, ZIP, and MP4 files to Zotero. The main PDF uses ACS's established `/doi/pdf/<DOI>` route.
- Cell Press saves metadata and modern SI files, deduplicates repeated page markup, and rewrites downloadable `mmc*` files to Elsevier's public CDN to avoid `www.cell.com` Cloudflare challenges.
- Cell Press personal-library transfer succeeded for `10.1016/j.cell.2026.05.012`: one citation and both SI PDFs were accepted by Zotero.
- Download and link-only translator modes passed for ACS and Nature attachment descriptors.
- The final non-live Connector suite passes: 103 passing, 4 opt-in live diagnostics skipped.

## What does not work yet

- The exact JACS SI PDF transfers successfully, but the isolated Chrome test profile cannot transfer the ACS article PDF because ACS returns HTTP 403 without the user's normal browser session. In normal Chrome, `/doi/pdf/10.1021/jacs.5c22031` redirects to and opens the full PDF. Load the unpacked build in that profile for the final combined-save proof.
- The Connector has a narrow temporary-user-agent rule for `ndownloader.figshare.com/files/`; it is required because Figshare's generic download hostname returns Amazon WAF HTTP 202 to Chrome's normal user agent. Do not broaden it to `pubs.acs.org`.
- Cell Press's main full-text PDF URL still receives a Cloudflare challenge in the isolated Connector browser. The tested citation received both SI PDFs but not the main PDF.
- The Zotero Desktop endpoint test is committed but was not executed successfully in this checkout. Its build scripts fail when the Desktop path contains spaces, and the CI configuration expects Node 18. Test it from a no-space checkout with the supported toolchain.
- ACS PDF/XLSX/ZIP/MP4 SI transfers all passed against live articles and the user's Zotero library. Broader issue-page selection still lacks a complete live multiple-item test.

## Safe tests for the next agent

From the Connector repository:

```sh
./build.sh -d
env HEADLESS=true npm test
```

Live translation without writing to a Zotero library:

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

Focused Cell Press binary-fetch diagnostic without a library write:

```sh
env LIVE_PUBLISHER_TESTS=true LIVE_ATTACHMENT_FETCH_ONLY=true \
  LIVE_CELL_PRESS_URL='https://www.cell.com/cell/fulltext/S0092-8674(26)00571-4' \
  npx mocha --grep 'Live publisher translator diagnostics Cell Press$' \
  --timeout 120000
```

Focused exact-JACS translation without a library write:

```sh
env LIVE_PUBLISHER_TESTS=true EXPECT_PUBLISHER_FIXES=true \
  npx mocha test/tests/livePublisherTest.mjs --grep 'ACS$' --timeout 120000
```

Translator syntax:

```sh
cd src/zotero/translators
node .bin/check-syntax.mjs 'ACS Publications.js'
node .bin/check-syntax.mjs 'Nature Publishing Group.js'
node .bin/check-syntax.mjs 'Cell Press.js'
```

From a no-space Zotero Desktop checkout using Node 18:

```sh
test/runtests.sh -b -g 'should return translator preferences to the Connector'
```

## Library-writing test warning

`LIVE_LIBRARY_TRANSFER=true` sends real writes to Zotero's currently selected library. Do not run it casually. Authorized runs left these citations in **My Library** for the user to remove later:

- Nature citation with the main PDF and all 21 SI files.
- Six ACS citations from the reassessment: two copies of the requested JACS item with its SI PDF, plus JPCA, JCIM benchmark, ES&T, and JCIM multimedia items with all expected SI files. No ACS test item received the main article PDF from the isolated browser profile.
- Cell Press citation `10.1016/j.cell.2026.05.012` with both SI PDFs. The user manually deleted the incomplete first attempt before the successful replacement save.

No test citation or attachment was deleted.
