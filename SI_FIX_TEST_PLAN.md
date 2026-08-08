# Supplementary Information Fix: Simple Test Plan

## What we need to prove

The Chrome Connector should use Zotero desktop's supplementary-file setting when Zotero is open. When Zotero is closed, it should go back to the Connector's own setting. Changing either setting must not require reinstalling the extension.

## Automated checks

1. Build the unpacked Chrome extension from a folder whose name contains spaces.
2. Run the complete Connector test suite.
3. Run the Zotero desktop connector-server test that checks the new `/connector/ping` preference payload.
4. Confirm that an older Zotero response without `translatorPrefs` still works.

## Hands-on Chrome checks

Load `build/manifestv3` as an unpacked extension, then test these cases:

| Zotero desktop | Desktop supplementary setting | Connector setting | Expected result |
| --- | --- | --- | --- |
| Open | On | Off or unset | Translator sees **On** |
| Open | Off | On | Translator sees **Off** |
| Closed | Not available | On | Translator sees **On** |
| Closed | Not available | Off | Translator sees **Off** |

Repeat the first case with:

- “Supplementary as link” off: files should download.
- “Supplementary as link” on: Zotero should save links without downloading the payloads.
- “Download associated files” off: binary attachments should not download.
- The desktop setting changed while Chrome remains open: the next save should use the new value.
- Zotero disconnected after a successful save: the next save should use the Connector-local value, not a stale desktop value.

## Publisher checks after their separate translator patches

### Nature

- `nature10840`: exactly one supplementary PDF.
- A modern article with source data: include the real PDF/XLSX files and exclude Extended Data figure pages.
- Link mode: the same files appear as links.

### Cell Press

- The issue's Heliyon article: exactly one supplementary PDF, with no duplicate from the side panel.
- A current Cell article: PDF and XLSX supplements receive the right names and file types.
- Unknown file types remain links and do not break the parent-item save.

### ACS

- A new Silverchair article URL is detected by the ACS translator.
- DOI, citation metadata, primary PDF, and supplementary resources all resolve from the new page.
- PDF, XLSX, ZIP, and MP4 supplementary resources are handled deliberately.
- A new issue page can still offer multiple item selection.
- Existing legacy ACS test cases continue to pass.

## Pass conditions

- The full Connector suite passes.
- The desktop endpoint test passes in Zotero's test runtime.
- Chrome receives the desktop preference before translation begins.
- Chrome's offscreen translator refreshes the value for every new save.
- Disconnecting Zotero removes desktop overrides without deleting Connector-local settings.
- A supplementary-file failure never prevents the citation and main PDF from saving.

## Validation completed in this workspace

- Debug Chrome build: passed.
- Full Connector suite: 102 tests passed after the disconnect-cleanup test was added.
- Focused Connector preference coverage: included in the passing full suite.
- Zotero desktop companion files: JavaScript syntax checks passed.
- Live patched translators in Chrome for Testing: ACS, Nature, and Cell Press passed in download mode.
- Requested ACS and Nature articles: passed in both download and link modes.
- Nature result: 3 PDF and 18 XLSX supplements, with all 10 figure-page false positives excluded.
- Zotero desktop endpoint test: added, but the application build is blocked by upstream scripts that do not quote this workspace path containing spaces.
