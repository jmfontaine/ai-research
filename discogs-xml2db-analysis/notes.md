# Research Notes

## Initial Setup
- Created analysis folder
- Cloned repository from https://github.com/philipmat/discogs-xml2db

## Key Files Analyzed

### Core Application
- `run.py` - Entry point with CLI using docopt
- `discogsxml2db/parser.py` - XML parsing logic using lxml's iterparse
- `discogsxml2db/exporter.py` - CSV export logic with bz2 compression support

### PostgreSQL Integration
- `postgresql/importcsv.py` - Uses psycopg2 COPY command for bulk import
- `postgresql/psql.py` - Wrapper for psql command-line tool
- `postgresql/dbconfig.py` - Configuration management
- `postgresql/sql/` - Schema files (CreateTables, CreatePrimaryKeys, CreateFKConstraints, CreateIndexes)

### MySQL Integration
- `mysql/importcsv.py` - Uses mysql.connector with LOAD DATA LOCAL INFILE
- `mysql/importcsv.sh` - Shell script with pv progress display
- `mysql/CreateTables.sql` - Schema using MyISAM engine
- `mysql/AssignPrimaryKeys.sql` - Primary key constraints

### Alternative Implementation
- `alternatives/dotnet/` - C# version with significant performance improvements

## Key Findings

### Parsing Approach
- Uses lxml.etree.iterparse for streaming XML parsing
- Memory-efficient: clears elements after processing
- Entity classes: Artist, Label, Master, Release

### Export Strategy
- Two-phase approach:
  1. Parse XML -> Export to CSV
  2. CSV -> Database (bulk load)
- Much faster than row-by-row database inserts

### Performance
- Can work with compressed .xml.gz files directly
- Supports bz2 compression for output CSV
- C# version is 2-3x faster than Python

### Data Model
- 4 main entity types: artist, label, master, release
- 28 total tables with normalized structure
- Genres, styles, images moved to separate tables in v2.0

### Limitations
- MySQL Python import doesn't support compressed files
- MySQL Python import shows no progress
- No indexes/foreign keys defined for MySQL
