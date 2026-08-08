# Live supplementary-information test results

Test date: 2026-08-08

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
- Produced one main PDF and exactly one supplementary PDF.
- Supplement URL: `/jacsat/article-supplement/5206327/pdf/ja5c22031_si_001/`.
- Download mode emitted `application/pdf` with `snapshot=true`.
- Link mode emitted the same file with `snapshot=false`.
- The old DOI/RIS request still returns HTTP 403 in Chrome, confirming that the metadata-based ACS implementation is necessary.

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
- Full Connector regression suite: **102 passing**, 4 live diagnostics skipped by default.
- Translator syntax checks and `git diff --check`: passed.

## Remaining end-to-end check

An explicitly authorized transfer test was subsequently run against the user's selected **My Library** destination. The imported items were left in place:

- Nature: `saveItems` succeeded, followed by 22 successful `saveAttachment` calls—one main PDF and all 21 supplementary files.
- ACS: `saveItems` succeeded, so the citation remains in My Library. ACS rejected both the main PDF and SI binary fetches before they could be sent to Zotero; its attachment-resolver fallback returned HTTP 500.
- A bounded ACS browser-challenge bypass probe also timed out. The experimental whitelist change was reverted because it did not solve the transfer failure.

## Cell Press personal-library transfer

[Deep-sea megafauna co-opts microbial energy metabolism genes to withstand ultra-long starvation](https://www.cell.com/cell/fulltext/S0092-8674(26)00571-4)

- Detected **Cell Press** and saved DOI `10.1016/j.cell.2026.05.012`.
- The original `www.cell.com/cms/.../attachment/...` URLs returned Cloudflare HTTP 403 responses even from the loaded article page.
- The translator now derives the corresponding public `ars.els-cdn.com` URLs from the article PII and `mmc*` filenames in download mode.
- Connector-level fetch validation downloaded both PDFs: 2,806,935 bytes and 9,350,052 bytes.
- The replacement library save produced one successful `saveItems` call and two successful `saveAttachment` calls, one for each SI PDF.
- The main full-text PDF remained blocked by Cell Press; the SI transfer itself is confirmed functional.
- The user deleted the incomplete first attempt before the successful replacement citation was written.

The remaining ACS release issue is therefore binary delivery/authentication, not translator detection or SI discovery.

The companion Zotero Desktop endpoint test is present, but its application build is currently blocked by upstream build scripts that do not quote workspace paths containing spaces. The Connector preference bridge itself is covered by the passing Connector tests.
