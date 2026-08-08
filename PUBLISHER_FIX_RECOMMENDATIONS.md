# Publisher-Specific Follow-up Recommendations

These changes should follow the universal preference bridge as independent translator commits.

## Nature Publishing Group

Recommended commit:

`Nature Publishing Group: Restore supplementary-file capture`

The existing translator still looks for legacy `#supplementary-information` markup. Current Nature pages expose direct file links under:

- `[data-test="supplementary-info"]`
- `[data-test="supp-item"]`
- `a[data-test="supp-info-link"]`

Filter out Extended Data figure pages that share the link selector. Prefer links whose label contains `(download TYPE)` and whose URL is a direct media/file URL. Add current Office MIME types such as DOCX and XLSX, and preserve the old parser as a fallback.

## Cell Press

Recommended commit:

`Cell Press: Restore supplementary file capture on modern article pages`

The existing translator searches obsolete `#aotftabs`, `#article_options`, and `#main_supp` structures. Current pages expose files under:

- `#supplementary-material .core-supplementary-material`
- `.core-link a[href]`
- `.core-description`

Prefer the main supplementary section and use the collateral side panel only as a fallback, or deduplicate by absolute URL. Infer the type from the `download` filename or URL. Keep the old parser as a fallback.

## ACS Publications

Recommended commit:

`ACS Publications: Support Silverchair pages and supplements`

ACS moved to Silverchair in July 2026, so this is broader than one supplementary selector. The follow-up needs to update:

- Translator URL targets for new article, abstract, issue, and search routes.
- DOI detection, preferring `meta[name="citation_doi"]`.
- Metadata extraction through the page's Embedded Metadata and citation meta tags. Live Chrome receives HTTP 403 from the new `/Citation/Download` endpoint, so it must not be the primary path.
- Primary PDF discovery through `meta[name="citation_pdf_url"]`.
- Issue-page result selectors.
- Supplement discovery through `a[data-doctype="dataSupplementDoc"]`.
- File-type extraction from `/article-supplement/{id}/{format}/...` URLs.
- Duplicate filtering and explicit handling of PDF, XLSX, ZIP, and MP4.

Because metadata capture and ordinary PDFs are also affected, split this into two commits if review size becomes unwieldy:

1. `ACS Publications: Update for Silverchair migration`
2. `ACS Publications: Add supplementary attachment coverage`
