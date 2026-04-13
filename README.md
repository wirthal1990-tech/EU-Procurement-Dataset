# EU Public Procurement Flat-File

![Records](https://img.shields.io/badge/Records-4,643,046-blue)
![Countries](https://img.shields.io/badge/Countries-33-green)
![CPV Codes](https://img.shields.io/badge/CPV_Codes-9,296-orange)
![Version](https://img.shields.io/badge/Version-1.1-brightgreen)
![License](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey)

## Pain Point

TED (Tenders Electronic Daily) publishes **740,000+ procurement notices per year** across the European Economic Area — but only as 75-column CSVs with inconsistent dates, denormalized codes, and duplicate entries across yearly exports.

**No official flat-file exists for KMU, PropTech, or consulting firms** who need clean, queryable procurement data.

This dataset solves that in a single Parquet download: **4.64M lot-award records** across **1.42M unique notices** (2019–2023), cleaned, deduplicated at lot level, and enriched with 9,296 CPV descriptions in English and German.

## Schema (v1.1 — 29 fields)

| # | Field | Type | Nullable | Description |
|---|-------|------|----------|-------------|
| 1 | `notice_id` | VARCHAR | No | Unique procurement notice ID |
| 2 | `lot_number` | INTEGER | No | Lot sequence within notice (1-based) |
| 3 | `notice_type` | VARCHAR | No | Notice type (CAN) |
| 4 | `publication_date` | DATE | Yes | TED publication date |
| 5 | `award_date` | DATE | Yes | Contract award date |
| 6 | `country_code` | VARCHAR(2) | Yes | Buyer country (ISO 3166-1) |
| 7 | `cpv_code` | VARCHAR(8) | Yes | 8-digit CPV code |
| 8 | `cpv_description_en` | VARCHAR | Yes | CPV description (English) |
| 9 | `cpv_description_de` | VARCHAR | Yes | CPV description (German) |
| 10 | `cpv_division` | VARCHAR(2) | Yes | CPV division (first 2 digits) |
| 11 | `cpv_division_description_en` | VARCHAR | Yes | Division description (English) |
| 12 | `cpv_division_description_de` | VARCHAR | Yes | Division description (German) |
| 13 | `lot_value_eur` | DOUBLE | Yes | Lot value in EUR |
| 14 | `total_contract_value_eur` | DOUBLE | Yes | Sum of all lot values per notice |
| 15 | `lots_in_notice` | INTEGER | Yes | Number of lots in this notice |
| 16 | `procedure_type` | VARCHAR | Yes | Procurement procedure code |
| 17 | `contract_type` | VARCHAR | Yes | Contract type (S/W/U) |
| 18 | `award_criteria` | VARCHAR | Yes | Award criteria |
| 19 | `buyer_name` | VARCHAR | Yes | Contracting authority |
| 20 | `buyer_type` | VARCHAR | Yes | Authority type code |
| 21 | `buyer_country` | VARCHAR(2) | Yes | Buyer country |
| 22 | `winner_name` | VARCHAR | Yes | Winning contractor |
| 23 | `winner_country` | VARCHAR(2) | Yes | Winner country |
| 24 | `winner_nuts` | VARCHAR | Yes | Winner NUTS region |
| 25 | `sme_flag` | BOOLEAN | Yes | Winner is SME? |
| 26 | `gpa_flag` | BOOLEAN | Yes | GPA coverage |
| 27 | `number_of_offers` | INTEGER | Yes | Offers received |
| 28 | `framework_agreement` | BOOLEAN | Yes | Framework agreement? |
| 29 | `data_source` | VARCHAR | No | Source file identifier |

## Quick Start

```sql
-- DuckDB: Top 10 lot awards by country and CPV
SELECT country_code, cpv_description_en,
       COUNT(*) as lot_awards,
       ROUND(AVG(lot_value_eur), 2) as avg_lot_value_eur,
       ROUND(SUM(lot_value_eur), 2) as total_value_eur
FROM 'eu_procurement_flat_v1.1.parquet'
WHERE publication_date >= '2022-01-01'
  AND lot_value_eur IS NOT NULL
GROUP BY 1, 2
ORDER BY 5 DESC
LIMIT 10;
```

```python
import duckdb
con = duckdb.connect()
df = con.execute("""
    SELECT winner_country,
           COUNT(*) as wins,
           SUM(lot_value_eur) as total_eur,
           AVG(lots_in_notice) as avg_lots
    FROM 'eu_procurement_flat_v1.1.parquet'
    WHERE sme_flag = true
    GROUP BY 1 ORDER BY 3 DESC LIMIT 10
""").fetchdf()
print(df)
```

## Sample Output

| notice_id | publication_date | country_code | cpv_code | cpv_description_en | contract_value_eur | winner_name |
|-----------|-----------------|--------------|----------|-------------------|-------------------|-------------|
| 2023587654 | 2023-11-15 | DE | 72000000 | IT services | 2,450,000.00 | SAP SE |
| 2022412389 | 2022-06-20 | FR | 45000000 | Construction work | 15,800,000.00 | Bouygues Construction |
| 2021298456 | 2021-09-03 | PL | 33000000 | Medical equipment | 890,500.00 | Medtronic Poland Sp. z o.o. |
| 2020189234 | 2020-04-12 | ES | 79000000 | Business services | 340,000.00 | Deloitte Advisory S.L. |
| 2019142144 | 2019-03-22 | CZ | 09123000 | Petroleum products | 1,929,742.73 | Pražská plynárenská, a.s. |

## Sources and License

**Data**: © European Union, 1998–2023. Published on [TED (Tenders Electronic Daily)](https://ted.europa.eu/).

**License**: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Attribution required.

**Source Dataset**: [TED CSV on EU Open Data Portal](https://data.europa.eu/data/datasets/ted-csv)

## Related: Ethno-API Case Study

This flat-file was produced as **Case Study 2** for the [Ethno-API Custom AI Data Engineering Service](https://ethno-api.com).

See also: [Case Study 1 — Ethnobotanical Compound Dataset](https://ethno-api.com)

## Built with AI Orchestration

This flat-file was built in **~45 minutes** via AI-orchestrated data pipeline — from raw EU Open Data to production-ready Parquet.

**Manual equivalent**: 40–80 hours of data engineering work (download coordination, schema mapping, date parsing, deduplication, CPV enrichment, quality validation).

Pipeline: URL Discovery → Bulk Download (5 ZIPs, 486 MB) → DuckDB Transformation → Cleaning Rules C1–C8 → CPV Enrichment → Deduplication → Parquet Export → Automated Verification (SC1–SC3).
