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
- ACS's new Silverchair URL is detected by the ACS translator. The correct DOI, metadata, main-PDF candidate, and one SI PDF candidate are emitted.
- Cell Press saves metadata and modern SI files, deduplicates repeated page markup, and rewrites downloadable `mmc*` files to Elsevier's public CDN to avoid `www.cell.com` Cloudflare challenges.
- Cell Press personal-library transfer succeeded for `10.1016/j.cell.2026.05.012`: one citation and both SI PDFs were accepted by Zotero.
- Download and link-only translator modes passed for ACS and Nature attachment descriptors.
- The final non-live Connector suite passes: 102 passing, 4 opt-in live diagnostics skipped.

## What does not work yet

- ACS binary transfer is still blocked by publisher anti-bot behavior. The citation saves to Zotero, but both the main PDF and SI fetch fail before `saveAttachment`; the resolver fallback returns HTTP 500.
- Adding `pubs.acs.org` to the generic browser-challenge whitelist did not help: both hidden-frame and window-monitor paths timed out. That experiment was reverted and must not be reintroduced without a different mechanism.
- Cell Press's main full-text PDF URL still receives a Cloudflare challenge in the isolated Connector browser. The tested citation received both SI PDFs but not the main PDF.
- The Zotero Desktop endpoint test is committed but was not executed successfully in this checkout. Its build scripts fail when the Desktop path contains spaces, and the CI configuration expects Node 18. Test it from a no-space checkout with the supported toolchain.
- Broader ACS issue-page selection and the PDF/XLSX/ZIP/MP4 matrix have focused helper coverage, but not a complete live Chrome matrix after the Silverchair migration.

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

`LIVE_LIBRARY_TRANSFER=true` sends real writes to Zotero's currently selected library. Do not run it casually. The authorized runs in this task left three citations in **My Library** for the user to remove later:

- Nature citation with the main PDF and all 21 SI files.
- ACS citation without binary attachments.
- Cell Press citation `10.1016/j.cell.2026.05.012` with both SI PDFs. The user manually deleted the incomplete first attempt before the successful replacement save.

No test citation or attachment was deleted.
