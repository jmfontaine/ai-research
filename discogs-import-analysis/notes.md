# Research Notes

## Initial Observations

- Repository is a fork of the original discogs-sql-importer project
- Licensed under GPL
- Uses Python 2.7 (explicitly stated in README)
- Primarily uses SAX parsing for XML processing

## Code Structure Analysis

### Main Entry Point
- `parse_discogs.py` - Central CLI script with argparse-based interface
- Supports 5 output modes: json, pgsql, pgdump, couch, mongo

### Entity Handlers
Each entity type has a dedicated SAX ContentHandler:
- `artist.py` - ArtistHandler
- `label.py` - LabelHandler
- `release.py` - ReleaseHandler
- `master.py` - MasterHandler

All handlers follow the same pattern:
1. Track element state with inElement dictionary
2. Buffer character data
3. Build model objects from parsed elements
4. Send complete objects to exporter

### Exporters
- `export_json.py` - Simple JSON console output (uses ujson if available)
- `export_postgres.py` - Direct PostgreSQL insertion using psycopg2
- `export_mongodb.py` - MongoDB with direct insert or file-based bulk import
- `export_couchdb.py` - CouchDB document storage

### Notable MongoDB Features
- Two import modes: direct (mongodb://) and file-based (file://)
- MD5 hash deduplication with `?uniq=md5` option
- Creates `.md5` files for differential imports

### Data Quality Filtering
All exporters support filtering by data_quality values:
- 'Needs Vote'
- 'Complete And Correct'
- 'Correct'
- 'Needs Minor Changes'
- 'Needs Major Changes'
- 'Entirely Incorrect'
- 'Entirely Incorrect Edit'

## Performance Observations
- README claims initial direct MongoDB import can "run for days"
- File-based JSON dump + mongoimport is "considerably faster"
- MD5 hashing enables differential imports (November to December example shows artists.json reduced from 554MB to 18MB)

## Limitations Found
- Python 2.7 only (syntax and module references confirm this)
- MySQL not supported
- No batch processing for PostgreSQL (one INSERT per record)
- PostgreSQL has no upsert support (initial import only)
- Some XML elements are commented out as unknown (videos, identifiers)
- Foreign key relationships use names instead of IDs in PostgreSQL

## Schema Analysis (PostgreSQL)
- 21 tables total
- Core tables: artist, label, release, master
- Junction tables for many-to-many relationships
- Arrays used for URLs, genres, styles, etc.
- Images stored separately with URI as primary key

## Utility Tools
1. `fix-xml.py` - Adds missing root tags and strips control characters
2. `get_latest_dumps.sh` - Downloads latest 4 dump files from discogs.com
3. `make_md5.py` - Generates MD5 checksums for change detection
4. `make_artist_name_id_pair.py` - Extracts artist name/ID pairs from JSON
