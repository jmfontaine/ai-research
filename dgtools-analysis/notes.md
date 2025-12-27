# Research Notes

## Initial Setup
- Created analysis folder: dgtools-analysis
- Cloned repository from https://github.com/marcw/dgtools

## Repository Structure
- Written in Go (go 1.25.0)
- Uses urfave/cli/v3 for CLI framework
- Main entry point: main.go
- Commands organized in cmd_*.go files
- Core parsing logic in internal/discogs/

## Key Files Analyzed
- `main.go` - Entry point, version 0.3.0
- `cmd_dump.go` - Dump command group
- `cmd_dump_list.go` - List S3 bucket contents
- `cmd_dump_download.go` - Download with checksum verification
- `cmd_dump_convert.go` - Convert to parquet/ndjson
- `cmd_dump_structure.go` - Analyze XML structure
- `cmd_db.go` - Database command group
- `cmd_db_prepare.go` - Run migrations with goose
- `cmd_db_import.go` - Import data using PostgreSQL COPY protocol
- `cmd_db_nuke.go` - Rollback migrations
- `internal/discogs/dump.go` - XML parsing and gzip decompression
- `internal/discogs/models.go` - Data models with parquet/json tags
- `internal/discogs/import.go` - PostgreSQL import logic with channels

## Key Dependencies
- github.com/parquet-go/parquet-go - Parquet file writing
- github.com/jackc/pgx/v5 - PostgreSQL driver with COPY support
- github.com/pressly/goose/v3 - Database migrations
- github.com/briandowns/spinner - Progress indicators
- github.com/charmbracelet/lipgloss - Terminal styling

## Data Model Observations
- 4 main entity types: Artist, Label, Master, Release
- Nested types: Name, Video, Track, Identifier, Company, etc.
- Struct tags for xml, json, parquet with compression options
- Uses zstd and dict compression in parquet

## Database Schema
- 10 PostgreSQL tables
- Main tables: discogs_artists, discogs_labels, discogs_masters, discogs_releases
- Join tables: discogs_artists_aliases, discogs_artists_members, discogs_master_artists,
  discogs_release_artists, discogs_release_extra_artists, discogs_release_labels
- Uses JSONB for nested arrays (urls, videos, genres, styles, tracklist, etc.)
- Extensive B-tree indexing for relationships

## Performance Techniques
- Single-pass parsing with channel-based distribution
- Uses PostgreSQL COPY protocol for bulk inserts
- Concurrent table loading with connection pool
- Buffered channels (1000 records) for backpressure control
- Spinner progress updates every 1000 records

## Limitations Identified
- PostgreSQL only for database imports (no MySQL, SQLite)
- Relies on sha256sum system command for checksum verification
- No explicit performance benchmarks published
- No Windows support in CI (Linux/Darwin only)
