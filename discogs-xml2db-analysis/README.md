# Analysis of discogs-xml2db Repository

**Repository:** https://github.com/philipmat/discogs-xml2db
**Analysis Date:** December 27, 2025

## Summary

discogs-xml2db is a Python tool for importing [Discogs data dumps](https://data.discogs.com/) into databases. Version 2 is a significant rewrite that is "several times faster" than the original, using a two-phase approach: first converting XML dumps to CSV files, then bulk-loading CSV files into databases.

---

## Deployment Options

The project offers two deployment options:

### 1. Python Version (Primary)
- Clone the repository and install dependencies via pip
- Requires Python 3.6+ and a virtual environment
- Platform-independent (Linux, macOS, Windows)

### 2. .NET Version (Alternative/Experimental)
- Pre-compiled binaries available on the [Releases page](https://github.com/philipmat/discogs-xml2db/releases)
- No installation required - download and run
- Available for multiple platforms (self-contained executables)
- Significantly faster than the Python version (2-3x performance improvement)

---

## Installation Requirements (Users)

### Python Version

**Required:**
- Python 3.6 or higher
- Bash shell (for automation scripts)
- Dependencies from `requirements.txt`:
  - `docopt` - Command-line argument parsing
  - `lxml` - XML parsing
  - `requests` - HTTP requests (for API counts)
  - `tqdm` - Progress bars

**PostgreSQL-specific:**
- `libpq-dev` (system library on Debian/Ubuntu)
- `psycopg2` (Python package from `postgresql/requirements.txt`)

**MySQL-specific:**
- `mysql-connector-python` (for Python import script)
- MySQL client tools (for shell import script)

**Optional:**
- `pv` (Pipe Viewer) - Displays progress during database import

### .NET Version
- No installation required - self-contained executables

---

## Available Commands

### Python Version (`run.py`)

```bash
python3 run.py [options] <INPUT_FILE>...
python3 run.py [options] INPUT_DIR --export=<entity>...
```

| Option | Description |
|--------|-------------|
| `--bz2` | Compress output CSV files using bz2 compression |
| `--limit=<lines>` | Limit export to specified number of entities |
| `--export=<entity>` | Select entity types: artist, label, master, release (repeatable) |
| `--debug` | Enable debug output |
| `--apicounts` | Get accurate counts from Discogs API for progress bar |
| `--dry-run` | Parse files without writing output |
| `--output=<dir>` | Output directory for CSV files (default: current directory) |

### Helper Scripts

| Script | Description |
|--------|-------------|
| `get_latest_dumps.sh` | Downloads latest Discogs dump files from S3 |
| `mysql/importcsv.sh` | Imports CSV files into MySQL (supports bz2) |
| `mysql/exec_sql.sh` | Executes SQL files against MySQL |
| `postgresql/psql.py` | Wrapper for psql command with config |
| `postgresql/importcsv.py` | Imports CSV files into PostgreSQL |

### .NET Version (`discogs`)

```bash
discogs [options] [files...]
```

| Option | Description |
|--------|-------------|
| `--dry-run` | Parse only, don't write files |
| `--verbose` | Enable verbose output |
| `--gz` | Compress output files with gzip |

---

## Project Strengths

1. **Performance-Optimized Architecture**: Two-phase approach (XML→CSV→Database) enables bulk loading, dramatically faster than row-by-row inserts

2. **Memory Efficient**: Uses streaming XML parsing (`iterparse`) that clears elements after processing, allowing processing of multi-GB files

3. **Works with Compressed Files**: Can read directly from `.xml.gz` dump files, reducing storage requirements (57GB uncompressed → 8.8GB compressed)

4. **Accurate Progress Reporting**: Real-time progress bars with optional API-based counts for precise estimates

5. **Database Agnostic**: CSV intermediate format allows easy adaptation to new databases

6. **Comprehensive Data Model**: Normalized schema with 28 tables covering all Discogs entity types

7. **Alternative High-Performance Implementation**: C# version provides 2-3x performance improvement

8. **Validation and Testing**: Includes test suite with expected record counts

---

## Known Limitations

1. **MySQL Python Import Script Limitations**:
   - Cannot import compressed CSV files
   - Does not display import progress
   - These limitations exist because MySQL requires file paths, not file objects

2. **MySQL Missing Features**:
   - No foreign key constraint scripts provided (only primary keys)
   - No index creation scripts

3. **PostgreSQL-Specific**:
   - Requires `libpq-dev` system dependency on Debian/Ubuntu

4. **.NET Version Incomplete Features**:
   - No `--api-counts` option for accurate progress
   - No direct database import (requires Python scripts)
   - No output folder specification (outputs to same folder as input)

5. **Platform Dependencies**:
   - Shell scripts require bash
   - Some automation not available on Windows without bash

---

## Performance Claims

The project makes explicit performance comparisons between Python and C# versions:

| File | Record Count | Python | C# |
|------|-------------:|:------:|:--:|
| discogs_20200806_artists.xml.gz | 7,046,615 | 6:22 | 2:35 |
| discogs_20200806_labels.xml.gz | 1,571,873 | 1:15 | 0:22 |
| discogs_20200806_masters.xml.gz | 1,734,371 | 3:56 | 1:57 |
| discogs_20200806_releases.xml.gz | 12,867,980 | 1:45:16 | 42:38 |

**Key performance characteristics:**
- C# version is approximately 2-3x faster than Python
- Architecture avoids database round-trips during parsing
- Bulk loading is dramatically faster than individual inserts
- Reduced disk space: compressed dumps (8.8GB) + compressed CSV (6.1GB) vs. 57GB uncompressed

---

## XML Parsing Implementation

The parsing is implemented in `discogsxml2db/parser.py` using **lxml's iterparse** for streaming XML processing.

### Core Parsing Pattern

```python
# From parser.py:80-91
def parse(self, fp):
    for event, element in etree.iterparse(fp, tag=self.entity_tag):
        i = self.entity_id(element)
        if i is not None:
            yield self.build_entity(i, element)
            element.clear()
            # Eliminate empty references from root node
            for ancestor in element.xpath('ancestor-or-self::*'):
                while ancestor.getprevious() is not None:
                    del ancestor.getparent()[0]
```

### Parser Classes

| Parser Class | Entity Tag | Description |
|--------------|------------|-------------|
| `DiscogsArtistParser` | `artist` | Parses artist records with aliases, URLs, groups, members |
| `DiscogsLabelParser` | `label` | Parses label records with sublabels, URLs |
| `DiscogsMasterParser` | `master` | Parses master releases with artists, genres, styles, videos |
| `DiscogsReleaseParser` | `release` | Parses releases with tracks, formats, identifiers, companies |

### Memory Management
- Elements are cleared immediately after processing (`element.clear()`)
- Previous siblings are deleted to prevent memory accumulation
- Allows processing of multi-GB XML files with minimal memory footprint

---

## Data Processing Implementation

Processing is implemented in `discogsxml2db/exporter.py` with specialized exporter classes.

### Processing Flow

1. **File Opening**: Detects `.gz` or `.xml` extension, opens appropriately
2. **Entity Parsing**: Iterates through parsed entities from the parser
3. **Validation**: Entity-specific validation (e.g., artists without names get placeholder)
4. **CSV Writing**: Executes configured write operations for each entity
5. **Progress Tracking**: Uses `tqdm` for real-time progress display

### Exporter Classes

| Exporter Class | Output Tables |
|----------------|---------------|
| `LabelExporter` | label, label_url, label_image |
| `ArtistExporter` | artist, artist_alias, artist_namevariation, artist_url, artist_image, group_member |
| `MasterExporter` | master, master_artist, master_video, master_genre, master_style, master_image |
| `ReleaseExporter` | release, release_genre, release_style, release_label, release_video, release_format, release_company, release_identifier, release_track, release_artist, release_track_artist, release_image |

### Write Operations

Three main write patterns:

1. **`_write_entity`**: Writes main entity fields to a single row
2. **`_write_rows`**: Writes simple parent-child relationships (e.g., URLs)
3. **`_write_fields_rows`**: Writes complex child entities with multiple fields

---

## Data Export Formats

### CSV Export (Primary)

**Location:** Output directory specified via `--output` flag

**Compression Options:**
- Uncompressed: `{table}.csv`
- Compressed: `{table}.csv.bz2` (with `--bz2` flag)
- .NET version: `{table}.csv.gz` (with `--gz` flag)

**Format Details:**
- Standard CSV with comma delimiter
- Fields optionally enclosed in double quotes
- Header row with column names
- UTF-8 encoding
- Windows-compatible line endings (`newline=''`)

**CSV Headers** (from `exporter.py`):

```python
csv_headers = {
    'label': ['id', 'name', 'contact_info', 'profile', 'parent_name', 'data_quality'],
    'artist': ['id', 'name', 'realname', 'profile', 'data_quality'],
    'master': ['id', 'title', 'year', 'main_release', 'data_quality'],
    'release': ['id', 'title', 'released', 'country', 'notes',
                'data_quality', 'master_id', 'status'],
    # ... plus all relationship tables
}
```

---

## Database Loading

### PostgreSQL

**Import Script:** `postgresql/importcsv.py`

**Method:** Uses psycopg2's `copy_expert` with `COPY ... FROM STDIN`:

```python
# From postgresql/importcsv.py:23-25
q = sql.SQL("COPY {} ({}) FROM STDIN WITH CSV HEADER").format(
    sql.Identifier(table),
    sql.SQL(', ').join(map(sql.Identifier, csv_headers[table])))
cursor.copy_expert(q, fp)
```

**Features:**
- Supports both uncompressed and bz2-compressed CSV files
- Reads column headers from shared `csv_headers` definition
- Configuration via `postgresql.conf` file

**Workflow:**
```bash
# 1. Create tables
python3 postgresql/psql.py < postgresql/sql/CreateTables.sql

# 2. Import CSV files
python3 postgresql/importcsv.py /csvdir/*

# 3. Add constraints and indexes
python3 postgresql/psql.py < postgresql/sql/CreatePrimaryKeys.sql
python3 postgresql/psql.py < postgresql/sql/CreateFKConstraints.sql
python3 postgresql/psql.py < postgresql/sql/CreateIndexes.sql
```

### MySQL

**Import Methods:**

1. **Shell Script** (`mysql/importcsv.sh`):
   - Supports both `.csv` and `.csv.bz2` files
   - Uses `LOAD DATA LOCAL INFILE` via stdin
   - Shows progress with `pv` if available

2. **Python Script** (`mysql/importcsv.py`):
   - Only supports uncompressed `.csv` files
   - Uses `mysql.connector` with `allow_local_infile=True`
   - No progress display

**MySQL LOAD DATA syntax:**
```sql
LOAD DATA LOCAL INFILE '/dev/stdin'
INTO TABLE `table_name`
FIELDS TERMINATED BY ',' ESCAPED BY '' OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\r\n'
IGNORE 1 LINES
(column1, column2, ...);
```

**Workflow:**
```bash
# 1. Create tables
mysql/exec_sql.sh < mysql/CreateTables.sql

# 2. Import CSV files
mysql/importcsv.sh /csvdir/*

# 3. Add primary keys
mysql/exec_sql.sh < mysql/AssignPrimaryKeys.sql
```

### MongoDB (Instructions Only)

Uses `mongoimport` command-line tool:
```bash
mongoimport --db=discogs --collection=releases \
            --type=csv --headerline --file=release.csv
```

### CouchDB (Instructions Only)

Requires converting CSV to JSON using `couchimport` tool.

---

## Data Model

### Entity Relationship Overview

The data model consists of 4 core entities with 28 total tables:

```
ARTIST (1) ──┬── (*) artist_alias
             ├── (*) artist_namevariation
             ├── (*) artist_url
             ├── (*) artist_image
             └── (*) group_member

LABEL (1) ───┬── (*) label_url
             ├── (*) label_image
             └── (0..1) parent_label [self-reference]

MASTER (1) ──┬── (*) master_artist
             ├── (*) master_video
             ├── (*) master_genre
             ├── (*) master_style
             └── (*) master_image

RELEASE (1) ─┬── (*) release_artist
             ├── (*) release_label
             ├── (*) release_track ──── (*) release_track_artist
             ├── (*) release_format
             ├── (*) release_genre
             ├── (*) release_style
             ├── (*) release_video
             ├── (*) release_identifier
             ├── (*) release_company
             └── (*) release_image
```

### Core Tables

| Table | Key Columns | Description |
|-------|-------------|-------------|
| `artist` | id, name, realname, profile, data_quality | Musicians, bands, DJs |
| `label` | id, name, contact_info, profile, parent_id, parent_name | Record labels |
| `master` | id, title, year, main_release, data_quality | Master releases (canonical version) |
| `release` | id, title, released, country, notes, master_id, status | Specific release versions |

### Relationship Tables

| Table | Foreign Keys | Description |
|-------|--------------|-------------|
| `artist_alias` | artist_id | Alternative artist names |
| `artist_namevariation` | artist_id | Name variations/spellings |
| `artist_url` | artist_id | Artist websites |
| `artist_image` | artist_id | Image metadata (type, dimensions) |
| `group_member` | group_artist_id, member_artist_id | Band membership |
| `label_url` | label_id | Label websites |
| `label_image` | label_id | Label image metadata |
| `master_artist` | master_id, artist_id | Master release credits |
| `master_video` | master_id | Video links (YouTube, etc.) |
| `master_genre` | master_id | Genre classifications |
| `master_style` | master_id | Style sub-classifications |
| `master_image` | master_id | Master release images |
| `release_artist` | release_id, artist_id | Release credits (with extra flag) |
| `release_label` | release_id, label_id | Label/catalog information |
| `release_track` | release_id | Track listings with sequence |
| `release_track_artist` | release_id, track_id, artist_id | Per-track credits |
| `release_format` | release_id | Physical/digital format details |
| `release_genre` | release_id | Release genre tags |
| `release_style` | release_id | Release style tags |
| `release_video` | release_id | Associated videos |
| `release_identifier` | release_id | Barcodes, catalog numbers |
| `release_company` | release_id, company_id | Credits (pressed by, etc.) |
| `release_image` | release_id | Release artwork metadata |

### Database-Specific Notes

**PostgreSQL:**
- Uses standard PostgreSQL types (integer, text, SERIAL)
- Full foreign key constraints defined
- Comprehensive index coverage
- Supports schema namespacing

**MySQL:**
- Uses MyISAM engine (faster for bulk imports, no FK support)
- SERIAL type for auto-increment
- Only primary key constraints defined
- No foreign key or index scripts provided

### Schema Changes from v1 (Classic)

Key changes in v2.0:
- Renamed tables for consistency (e.g., `releases_labels` → `release_label`)
- Renamed columns for clarity (e.g., `join_relation` → `join_string`)
- Added `parent_id` to `label` table
- Moved multi-valued fields to separate tables (genres, styles, aliases, URLs)
- Unified extra artists into main artist tables with `extra` flag
- Added `track_id` for track-artist relationships

---

## Conclusion

discogs-xml2db is a well-architected tool for importing Discogs data dumps into relational databases. Its two-phase design (XML→CSV→Database) provides significant performance benefits over direct database inserts. The codebase demonstrates good practices for processing large XML files with memory efficiency, and the normalized data model provides flexibility for various query patterns.

The Python version is production-ready with comprehensive features, while the experimental C# version offers substantial performance improvements for users who prioritize speed. PostgreSQL integration is more complete than MySQL, with full constraint and index support.
