# Live supplementary-information test results

Test date: 2026-08-08

## Required runtime configuration

- In Zotero Desktop's Config Editor, set `extensions.zotero.translators.attachSupplementary` to `true`.
- For downloaded files rather than link-only attachments, keep `extensions.zotero.translators.supplementaryAsLink` set to `false`.
- The patched Zotero Desktop build is required. Its `/connector/ping` response supplies `translatorPrefsVersion: 1` and the translator preferences to Chrome.
- In the Connector's own Config Editor, the equivalent local fallback key is `translators.attachSupplementary` (without the `extensions.zotero.` prefix).
- A stock Zotero Desktop build does not provide the preference handoff and may also serve the older publisher translators.

## How the test was run

- Built the modified Manifest V3 Connector with `./build.sh -d`.
- Loaded that build in Chrome for Testing 150.
- Seeded the patched ACS, Nature, and Cell Press translators from this workspace.
- Applied Zotero Desktop's effective settings in both translation modes:
  - download mode: `attachSupplementary=true`, `supplementaryAsLink=false`
  - link mode: `attachSupplementary=true`, `supplementaryAsLink=true`
- Stubbed only the final `saveItems` call, so live detection and translation ran without writing test items or attachments into the user's Zotero library.

## Requested ACS article

[Structural Engineering of Cyanine Dyes to Access Shortwave Infrared-Emissive J-Aggregates](https://pubs.acs.org/jacsat/article/148/28/29684/5206327/Structural-Engineering-of-Cyanine-Dyes-to-Access)

- Detected **ACS Publications** as the preferred translator.
- Saved DOI `10.1021/jacs.5c22031`.
- Produced one main PDF candidate and exactly one supplementary PDF.
- Restored the established main-PDF route: `/doi/pdf/10.1021/jacs.5c22031`.
- Resolved the supplementary file through ACS Figshare: `https://ndownloader.figshare.com/files/66503412`.
- Download mode emitted `application/pdf` with `snapshot=true`.
- Link mode emitted the same file with `snapshot=false`.
- The revised Connector downloaded all 9,911,839 SI bytes and transferred the PDF to Zotero.
- The original ACS PDF route redirects to the current Silverchair PDF and opens successfully in the user's normal Chrome session. A clean isolated Chrome profile receives HTTP 403 because it lacks that ACS browser session.

## Requested Nature article

[A dependency map enhanced with next-generation 3D cancer models](https://www.nature.com/articles/s41586-026-10843-7)

- Detected **Nature Publishing Group** as the preferred translator.
- Saved DOI `10.1038/s41586-026-10843-7`.
- Produced one main PDF plus exactly **21** supplementary files:
  - 3 PDFs
  - 18 XLSX spreadsheets
- Excluded all 10 Extended Data figure-page links that share Nature's supplementary-link selector.
- Download mode emitted downloadable attachments with the correct MIME types.
- Link mode emitted the same 21 URLs with `snapshot=false`.

## Other validation

- Patched Cell Press live article: one unique SI PDF, no duplicate, and the main item no longer fails on the obsolete PDF probe.
- Full Connector regression suite: **103 passing**, 4 live diagnostics skipped by default.
- Translator syntax checks and `git diff --check`: passed.

## Remaining end-to-end check

An explicitly authorized transfer test was subsequently run against the user's selected **My Library** destination. The imported items were left in place:

- Nature: `saveItems` succeeded, followed by 22 successful `saveAttachment` calls—one main PDF and all 21 supplementary files.
- ACS SI transfer now succeeds through Figshare. Authorized personal-library runs transferred the requested JACS SI PDF plus PDF, XLSX, ZIP, and MP4 supplements from four additional ACS articles.
- The exact JACS article's main PDF still cannot be transferred from the isolated Chrome test profile because ACS returns HTTP 403 there. The same restored `/doi/pdf/<DOI>` route opens the article PDF in the user's normal signed-in Chrome; the remaining combined-save check must use the unpacked build in that profile.

## Cell Press personal-library transfer

[Deep-sea megafauna co-opts microbial energy metabolism genes to withstand ultra-long starvation](https://www.cell.com/cell/fulltext/S0092-8674(26)00571-4)

- Detected **Cell Press** and saved DOI `10.1016/j.cell.2026.05.012`.
- The original `www.cell.com/cms/.../attachment/...` URLs returned Cloudflare HTTP 403 responses even from the loaded article page.
- The translator now derives the corresponding public `ars.els-cdn.com` URLs from the article PII and `mmc*` filenames in download mode.
- Connector-level fetch validation downloaded both PDFs: 2,806,935 bytes and 9,350,052 bytes.
- The replacement library save produced one successful `saveItems` call and two successful `saveAttachment` calls, one for each SI PDF.
- The main full-text PDF remained blocked by Cell Press; the SI transfer itself is confirmed functional.
- The user deleted the incomplete first attempt before the successful replacement citation was written.

The remaining ACS release check is one combined save from the unpacked Connector in the user's normal Chrome profile: the SI path is confirmed, and the restored article-PDF path is confirmed separately in that profile.

The companion Zotero Desktop endpoint test is present, but its application build is currently blocked by upstream build scripts that do not quote workspace paths containing spaces. The Connector preference bridge itself is covered by the passing Connector tests.
