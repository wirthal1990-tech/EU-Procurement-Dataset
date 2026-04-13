# Methodology — EU Public Procurement Flat-File v1.1

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-04-08 | Initial release — dedup by notice_id collapsed lots |
| 1.1 | 2026-04-08 | Lot-aware rebuild: PK (notice_id, lot_number), code-level CPV (9,296 codes), dedup on (notice_id, award_id) |

## Data Sources

### TED CSV — Contract Award Notices (2019–2023)
- **Source**: EU Open Data Portal — TED CSV Dataset
- **URL**: https://data.europa.eu/data/datasets/ted-csv
- **Format**: ZIP-compressed CSV, one file per year (2019–2023)
- **Fields**: 75 raw fields per notice, covering notice IDs, dates, CPV codes, contract values, buyer/winner info, SME flags, GPA coverage, and more
- **License**: CC BY 4.0 | © European Union, ted.europa.eu
- **Coverage**: ~990k–1.16M records per year, totaling 5.39M raw rows

### CPV Lookup (Common Procurement Vocabulary)
- **Source (v1.1)**: EU Publications Office SPARQL endpoint — 9,296 unique 8-digit CPV codes with EN + DE descriptions
- **Endpoint**: `https://publications.europa.eu/webapi/rdf/sparql`
- **Fallback**: 45-division CSV lookup for codes without SPARQL match
- **Usage**: Primary join on full 8-digit CPV code; fallback join on 2-digit division

### Why not BAFA?
BAFA (Bundesamt für Wirtschaft und Ausfuhrkontrolle) does not publish structured, machine-readable procurement or subsidy data. Their public outputs are PDF reports and aggregated statistics, making them unsuitable as a data source for this pipeline.

## Why Bulk Download over API

| Method | Estimated Time | Records | Risk |
|--------|---------------|---------|------|
| CSV Bulk Download (5 years) | ~15 min download + ~2 min processing | 5.39M raw | None (public, no auth) |
| TED REST API equivalent | 8–20h with pagination | Same | Rate limiting, timeout |
| XML Bulk (monthly packages) | ~60 min download + 3–5h processing | Comparable | Complex XML parsing |
| XML via API | >40h | Same | Impractical |

**Decision**: CSV Bulk Download chosen for 75% time savings over API approaches.

## Cleaning Rules

### C1: lot_value_eur / total_contract_value_eur (v1.1)
- Source field: `VALUE_EURO`
- `lot_value_eur`: per-row cleaned value (negative → NULL, > 1 trillion → NULL)
- `total_contract_value_eur`: SUM(lot_value_eur) OVER (PARTITION BY notice_id) — notice-level aggregate
- `TRY_CAST(VALUE_EURO AS DOUBLE)` for fault-tolerant conversion

### C2: cpv_code + cpv_description (v1.1)
- Source field: `CPV`
- Normalized to 8 digits: `LPAD(digits_only, 8, '0')`
- Check digit (after hyphen) removed; `cpv_division` = first 2 digits
- **v1.1**: cpv_description_en/de from code-level lookup (9,296 codes), with division-level fallback
- **v1.1**: cpv_division_description_en/de from division-level lookup (45 divisions)

### C3: country_code
- Source field: `ISO_COUNTRY_CODE`
- ISO 3166-1 Alpha-2: `UPPER(TRIM(...))`; length ≠ 2 → NULL

### C4: publication_date, award_date
- Source fields: `DT_DISPATCH`, `DT_AWARD`
- Format: `DD/MM/YY` → `TRY_STRPTIME(field, '%d/%m/%y')`

### C5: sme_flag
- Source field: `B_CONTRACTOR_SME`
- 'Y' → TRUE, 'N' → FALSE, else → NULL

### C6: winner_name, buyer_name
- Source fields: `WIN_NAME`, `CAE_NAME`
- `TRIM()` applied; empty strings → NULL

### C7: Deduplication (v1.1 — FIXED)
- **v1.0 bug**: Deduped by `notice_id` alone, collapsing multiple lots/awards per notice → only 1.42M rows
- **v1.1 fix**: Two-stage dedup:
  1. **Within-year**: Dedup on (notice_id, ID_AWARD) for rows with award IDs; DISTINCT for rows without. This removes true duplicates from TED's comma-separated lot-ID expansion while preserving multi-lot and multi-winner records.
  2. **Cross-year**: Dedup on (notice_id, lot_number) keeping newest publication_date.
- **lot_number**: Assigned via `ROW_NUMBER() OVER (PARTITION BY notice_id ORDER BY lot_value_eur DESC NULLS LAST, winner_name NULLS LAST)`
- **lots_in_notice**: `COUNT(*) OVER (PARTITION BY notice_id)`
- Result: 5.39M raw → 4,643,046 lot-award records (1,423,960 unique notices, avg 3.26 lots/notice)

### C8: Encoding
- CSV read with `all_varchar=true`, `ignore_errors=true`
- DuckDB handles UTF-8 natively
- `TRY_CAST` / `TRY_STRPTIME` used throughout

## Schema Design (v1.1)

29 fields in the output flat-file (28 + data_source). One row = one lot-award event within a notice.

| # | Field | Type | Nullable | Description |
|---|-------|------|----------|-------------|
| 1 | notice_id | VARCHAR | No | Unique procurement notice ID |
| 2 | lot_number | INTEGER | No | Lot sequence within notice (1-based) |
| 3 | notice_type | VARCHAR | No | Notice type (CAN) |
| 4 | publication_date | DATE | Yes | TED publication date |
| 5 | award_date | DATE | Yes | Contract award date |
| 6 | country_code | VARCHAR(2) | Yes | Buyer country (ISO 3166-1) |
| 7 | cpv_code | VARCHAR(8) | Yes | 8-digit CPV code |
| 8 | cpv_description_en | VARCHAR | Yes | CPV description (English, code-level) |
| 9 | cpv_description_de | VARCHAR | Yes | CPV description (German, code-level) |
| 10 | cpv_division | VARCHAR(2) | Yes | CPV division (first 2 digits) |
| 11 | cpv_division_description_en | VARCHAR | Yes | Division description (English) |
| 12 | cpv_division_description_de | VARCHAR | Yes | Division description (German) |
| 13 | lot_value_eur | DOUBLE | Yes | Lot value in EUR |
| 14 | total_contract_value_eur | DOUBLE | Yes | Sum of all lot values per notice |
| 15 | lots_in_notice | INTEGER | Yes | Number of lots in this notice |
| 16 | procedure_type | VARCHAR | Yes | Procurement procedure code |
| 17 | contract_type | VARCHAR | Yes | Contract type (S/W/U) |
| 18 | award_criteria | VARCHAR | Yes | Award criteria |
| 19 | buyer_name | VARCHAR | Yes | Contracting authority |
| 20 | buyer_type | VARCHAR | Yes | Authority type code |
| 21 | buyer_country | VARCHAR(2) | Yes | Buyer country |
| 22 | winner_name | VARCHAR | Yes | Winning contractor |
| 23 | winner_country | VARCHAR(2) | Yes | Winner country |
| 24 | winner_nuts | VARCHAR | Yes | Winner NUTS region |
| 25 | sme_flag | BOOLEAN | Yes | Winner is SME? |
| 26 | gpa_flag | BOOLEAN | Yes | GPA coverage |
| 27 | number_of_offers | INTEGER | Yes | Offers received |
| 28 | framework_agreement | BOOLEAN | Yes | Framework agreement? |
| 29 | data_source | VARCHAR | No | Source file identifier |

## Known Limitations

1. **eForms XML-Notices (from Nov 2022)**: Not processed. CSV covers the same period.
2. **BAFA**: No structured data available.
3. **TED CSV coverage**: CAN only. CN available but not included.
4. **Sub-threshold procurements**: Below EU thresholds (€143k supplies/services, €5.538M works) underrepresented.
5. **Winner name normalization**: Basic TRIM only — no entity resolution.
6. **Date format**: DD/MM/YY with DuckDB's default two-digit year interpretation.
7. **Lot granularity (v1.1)**: TED CSVs do not consistently populate ID_LOT_AWARDED. For rows without award IDs, lot_number is assigned by ROW_NUMBER() ordering, which is deterministic but arbitrary. Framework agreements often have many award rows per lot.

## Reproducibility

### Environment
- **Python**: 3.12.3 (`/usr/bin/python3`)
- **DuckDB**: 1.5.0
- **PyArrow**: 23.0.1
- **Pandas**: 2.3.3
- **OS**: Ubuntu Linux (Hetzner Cloud)

### Reproduction Steps
```bash
pip install duckdb pyarrow pandas requests tqdm --break-system-packages
python3 scripts/01a_discover_ted_csv_urls.py
python3 scripts/01b_download_ted_csv.py
python3 scripts/FIX_01_get_cpv_codes.py      # v1.1: code-level CPV
python3 scripts/FIX_02_rebuild_lot_aware.py   # v1.1: lot-aware rebuild
python3 scripts/FIX_03_regenerate_outputs.py  # v1.1: Parquet/JSON/MANIFEST
python3 scripts/FIX_04_sc3_final.py           # v1.1: SC3 verification
```

### Data Provenance
All source data is publicly available under CC BY 4.0 from the EU Open Data Portal.
