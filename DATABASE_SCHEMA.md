# Database Schema Documentation

## Table of Contents

1. [Overview](#overview)
2. [Bronze Layer Tables](#bronze-layer-tables)
   - [steam_game_bz](#steam_game_bz)
3. [Silver Layer Tables](#silver-layer-tables)
   - [steam_game_sl](#steam_game_sl)
4. [Gold Layer Tables](#gold-layer-tables)
   - [fact_steam_games_gl](#fact_steam_games_gl)
   - [dim_genres_gl](#dim_genres_gl)
   - [dim_categories_gl](#dim_categories_gl)
   - [dim_developers_gl](#dim_developers_gl)
   - [dim_publishers_gl](#dim_publishers_gl)
5. [Data Quality Tables](#data-quality-tables)
   - [data_quality_quarantine](#data_quality_quarantine)
6. [Analytics Tables](#analytics-tables)
   - [Date_Table](#date_table)
7. [Table Relationships](#table-relationships)

---

## Overview

This document describes the database schema for the Steam Games Analysis Project. The project follows a medallion architecture pattern with three layers: Bronze (raw data), Silver (cleaned data), and Gold (curated data for analytics). The schema is implemented using Delta Lake format on Azure Synapse Analytics.

**Database Name:** SteamDB  
**Storage Format:** Delta Lake  
**Platform:** Azure Synapse Analytics / Microsoft Fabric

---

## Bronze Layer Tables

### steam_game_bz

**Description:** Bronze layer table containing raw Steam game data ingested from CSV files. This table stores unprocessed data with minimal transformations, preserving the original source structure.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/medallion/bronze/steam_game_bz/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| appid | integer | Yes | Steam application ID - unique identifier for each game |
| name | string | Yes | Game name/title |
| release_year | integer | Yes | Year when the game was released |
| release_date | string | Yes | Release date in original format (may vary: "Jul 5, 2024", "Q4 2025", "2025", etc.) |
| genres | string | Yes | Semicolon-separated list of game genres (e.g., "Action;Adventure;RPG") |
| categories | string | Yes | Semicolon-separated list of game categories (e.g., "Single-player;Multi-player") |
| price | double | Yes | Game price in USD |
| recommendations | integer | Yes | Number of user recommendations/positive reviews |
| developer | string | Yes | Semicolon-separated list of game developers |
| publisher | string | Yes | Semicolon-separated list of game publishers |
| load_time | timestamp | Yes | Timestamp when the record was loaded into the bronze layer |
| source_file | string | Yes | Path to the source CSV file from which the record was ingested |

**Notes:**
- This is the first layer in the medallion architecture
- Data is ingested via streaming from raw CSV files
- All columns are nullable to handle missing data
- `release_date` is stored as string to preserve original format before parsing
- `genres`, `categories`, `developer`, and `publisher` may contain multiple values separated by semicolons

---

## Silver Layer Tables

### steam_game_sl

**Description:** Silver layer table containing cleaned and validated Steam game data. This table applies data quality checks, date parsing, and null handling transformations.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/medallion/silver/steam_game_sl/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| appid | integer | Yes | Steam application ID - unique identifier for each game (primary key) |
| name | string | Yes | Game name/title (nulls replaced with 'Unknown') |
| release_year | integer | Yes | Year when the game was released (nulls replaced with 0) |
| release_date | date | Yes | Parsed release date (standardized from various formats) |
| genres | string | Yes | Semicolon-separated list of game genres |
| categories | string | Yes | Semicolon-separated list of game categories |
| price | double | Yes | Game price in USD (nulls replaced with 0) |
| recommendations | integer | Yes | Number of user recommendations (nulls replaced with 0) |
| developer | string | Yes | Semicolon-separated list of game developers (nulls replaced with 'Unknown') |
| publisher | string | Yes | Semicolon-separated list of game publishers (nulls replaced with 'Unknown') |
| load_time | timestamp | Yes | Timestamp when the record was loaded into the bronze layer |
| source_file | string | Yes | Path to the source CSV file |
| update_time | timestamp | Yes | Timestamp when the record was last updated in the silver layer |

**Notes:**
- Data quality transformations applied:
  - Duplicate removal
  - All-null row removal
  - Null value handling (strings → 'Unknown', numbers → 0)
  - Date parsing from various formats (e.g., "Jul 5, 2024", "Q4 2025", "2025")
- Uses MERGE operation for upserts based on `appid`
- `release_date` is converted from string to date type
- Watermark is set on `load_time` for streaming processing

---

## Gold Layer Tables

### fact_steam_games_gl

**Description:** Fact table in the gold layer containing curated game information for analytics. This is the central fact table in the star schema.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/medallion/gold/fact_steam_games_gl/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| appid | integer | Yes | Steam application ID - unique identifier for each game (primary key, foreign key to dimension tables) |
| name | string | Yes | Game name/title |
| release_year | integer | Yes | Year when the game was released |
| release_date | date | Yes | Parsed release date |
| price | double | Yes | Game price in USD |
| recommendations | integer | Yes | Number of user recommendations/positive reviews |
| update_time | timestamp | Yes | Timestamp when the record was last updated in the gold layer |

**Notes:**
- This is the fact table in the star schema design
- Contains only game-level metrics and attributes
- Related to dimension tables via `appid`
- Used for analytics and reporting in Power BI
- Supports direct query mode in Microsoft Fabric

**Relationships:**
- One-to-many with `dim_genres_gl` (via `appid`)
- One-to-many with `dim_categories_gl` (via `appid`)
- One-to-many with `dim_developers_gl` (via `appid`)
- One-to-many with `dim_publishers_gl` (via `appid`)
- Many-to-one with `Date_Table` (via `release_date`)

---

### dim_genres_gl

**Description:** Dimension table containing game genres. Each game can have multiple genres, resulting in multiple rows per game.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/medallion/gold/dim_genres_gl/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| appid | integer | Yes | Steam application ID - foreign key to fact_steam_games_gl |
| genre | string | Yes | Individual genre name (extracted from semicolon-separated list) |
| update_time | timestamp | Yes | Timestamp when the record was last updated |

**Notes:**
- Created by exploding the `genres` column from the silver layer
- Composite primary key: (`appid`, `genre`)
- Empty genre strings are filtered out
- Used for filtering and grouping in analytics

**Relationships:**
- Many-to-one with `fact_steam_games_gl` (via `appid`)

---

### dim_categories_gl

**Description:** Dimension table containing game categories. Each game can have multiple categories, resulting in multiple rows per game.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/medallion/gold/dim_categories_gl/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| appid | integer | Yes | Steam application ID - foreign key to fact_steam_games_gl |
| category | string | Yes | Individual category name (extracted from semicolon-separated list) |
| update_time | timestamp | Yes | Timestamp when the record was last updated |

**Notes:**
- Created by exploding the `categories` column from the silver layer
- Composite primary key: (`appid`, `category`)
- Empty category strings are filtered out
- Examples: "Single-player", "Multi-player", "Steam Achievements", etc.

**Relationships:**
- Many-to-one with `fact_steam_games_gl` (via `appid`)

---

### dim_developers_gl

**Description:** Dimension table containing game developers. Each game can have multiple developers, resulting in multiple rows per game.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/medallion/gold/dim_developers_gl/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| appid | integer | Yes | Steam application ID - foreign key to fact_steam_games_gl |
| developer | string | Yes | Individual developer name (extracted from semicolon-separated list) |
| update_time | timestamp | Yes | Timestamp when the record was last updated |

**Notes:**
- Created by exploding the `developer` column from the silver layer
- Composite primary key: (`appid`, `developer`)
- Used for analyzing games by developer

**Relationships:**
- Many-to-one with `fact_steam_games_gl` (via `appid`)

---

### dim_publishers_gl

**Description:** Dimension table containing game publishers. Each game can have multiple publishers, resulting in multiple rows per game.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/medallion/gold/dim_publishers_gl/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| appid | integer | Yes | Steam application ID - foreign key to fact_steam_games_gl |
| publisher | string | Yes | Individual publisher name (extracted from semicolon-separated list) |
| update_time | timestamp | Yes | Timestamp when the record was last updated |

**Notes:**
- Created by exploding the `publisher` column from the silver layer
- Composite primary key: (`appid`, `publisher`)
- Used for analyzing games by publisher

**Relationships:**
- Many-to-one with `fact_steam_games_gl` (via `appid`)

---

## Data Quality Tables

### data_quality_quarantine

**Description:** Quarantine table for storing records that fail data quality validation rules. This table is used by Great Expectations for data quality monitoring.

**Storage Location:** `abfss://steam-game-project@dataprojectsforhuilu.dfs.core.windows.net/gx_config/data_quality/`

**Table Type:** Delta Lake

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| table_name | string | Yes | Name of the source table where the violation occurred |
| batch_id | long | Yes | Batch identifier for the data batch that failed validation |
| violated_rules | string | Yes | Description of the data quality rule that was violated |
| raw_data | string | Yes | The raw data record that failed validation (stored as string) |
| ingestion_time | timestamp | Yes | Timestamp when the record was quarantined |

**Notes:**
- Used for data quality monitoring and debugging
- Records that fail validation are stored here instead of being processed
- Supports schema validation, count checks, and custom Great Expectations rules
- Helps identify data quality issues in the pipeline

---

## Analytics Tables

### Date_Table

**Description:** Calculated date dimension table for time-based analytics in Power BI. This table provides various date hierarchies and attributes for reporting.

**Table Type:** Calculated (DAX) - Import Mode

| Column Name | Data Type | Nullable | Description |
|------------|-----------|----------|-------------|
| Date | date | No | Primary date column (date key) |
| Year | integer | No | Year extracted from date (e.g., 2024) |
| Quarter | string | No | Quarter in format "Q1", "Q2", "Q3", "Q4" |
| MonthNumber | integer | No | Month number (1-12) |
| MonthName | string | No | Full month name (e.g., "January", "February") |
| YearMonth | string | No | Year-month in format "YYYY-MM" (e.g., "2024-07") |
| DayName | string | No | Full day name (e.g., "Monday", "Tuesday") |
| DayOfWeekNumber | integer | No | Day of week number (1=Monday, 7=Sunday) |
| StartOfYear | date | No | First day of the year for the given date |
| StartOfMonth | date | No | First day of the month for the given date |
| StartOfWeek | date | No | First day of the week (Monday) for the given date |

**Date Range:** 2021-01-01 to 2030-12-31

**Notes:**
- Calculated table using DAX in Power BI
- Used for time-based filtering and grouping in reports
- Contains date hierarchy: Year → Quarter → Month → Date
- Multiple date variations for different analysis needs
- Related to `fact_steam_games_gl` via `release_date`

**Relationships:**
- One-to-many with `fact_steam_games_gl` (via `release_date`)

---

## Table Relationships

### Star Schema Design

The gold layer follows a star schema design with:

- **Fact Table:** `fact_steam_games_gl`
- **Dimension Tables:**
  - `dim_genres_gl`
  - `dim_categories_gl`
  - `dim_developers_gl`
  - `dim_publishers_gl`
  - `Date_Table` (time dimension)

### Relationship Details

1. **fact_steam_games_gl ↔ dim_genres_gl**
   - Relationship: One-to-Many
   - Join Key: `appid`
   - Cross-filtering: Both directions

2. **fact_steam_games_gl ↔ dim_categories_gl**
   - Relationship: One-to-Many
   - Join Key: `appid`
   - Cross-filtering: Both directions

3. **fact_steam_games_gl ↔ dim_developers_gl**
   - Relationship: One-to-Many
   - Join Key: `appid`
   - Cross-filtering: Both directions

4. **fact_steam_games_gl ↔ dim_publishers_gl**
   - Relationship: One-to-Many
   - Join Key: `appid`
   - Cross-filtering: Both directions

5. **fact_steam_games_gl ↔ Date_Table**
   - Relationship: Many-to-One
   - Join Key: `release_date` → `Date`
   - Join Behavior: Date part only

---

## Data Flow Summary

1. **Raw Data** → CSV files in `raw/` folder
2. **Bronze Layer** → `steam_game_bz` (streaming ingestion)
3. **Silver Layer** → `steam_game_sl` (data quality, cleaning, date parsing)
4. **Gold Layer** → Star schema tables:
   - `fact_steam_games_gl` (fact table)
   - `dim_genres_gl`, `dim_categories_gl`, `dim_developers_gl`, `dim_publishers_gl` (dimension tables)
5. **Analytics** → Power BI reports using gold layer tables

---

## Notes

- All tables use Delta Lake format for ACID transactions and time travel
- Streaming processing is used for bronze and silver layers
- MERGE operations are used for upserts in silver and gold layers
- Data quality validation is performed using Great Expectations
- The schema supports both batch and streaming data processing
- Tables are stored in Azure Data Lake Storage Gen2 (ADLS Gen2)

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Maintained By:** Data Engineering Team

