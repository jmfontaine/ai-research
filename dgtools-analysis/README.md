# dgtools Repository Analysis

This document provides a comprehensive analysis of the [dgtools](https://github.com/marcw/dgtools) repository, a command-line utility for working with Discogs data dumps.

## Overview

**dgtools** is a Go-based CLI tool (version 0.3.0) that simplifies working with the Discogs music database data dumps. It handles downloading, converting, and importing these large XML datasets into PostgreSQL databases or other formats.

---

## Deployment Options

### Pre-built Binaries (GitHub Releases)

The project provides pre-built binaries via GitHub Actions for the following platforms:

| Platform | Architecture |
|----------|-------------|
| Linux    | amd64       |
| Linux    | arm64       |
| macOS    | amd64       |
| macOS    | arm64       |

Binaries are automatically built and attached to GitHub releases when version tags are pushed.

### Build from Source

Users can compile the tool using Go 1.25+:

```bash
go build -o dgtools .
```

---

## Installation Requirements (Users)

### Minimum Requirements

1. **For pre-built binaries:**
   - Linux (amd64 or arm64) or macOS (amd64 or arm64)
   - `sha256sum` utility available in PATH (for checksum verification during downloads)

2. **For building from source:**
   - Go 1.25.0 or later

3. **For database import functionality:**
   - PostgreSQL server (accessible via connection URL)
   - A database created and accessible by the user

### No Windows Support

The CI/CD pipeline does not build Windows binaries, and the checksum verification relies on the `sha256sum` command which is not natively available on Windows.

---

## Available Commands

### Global Options

| Option | Description | Default |
|--------|-------------|---------|
| `--discogs-bucket` | URL of the Discogs data dumps S3 bucket | `https://discogs-data-dumps.s3.us-west-2.amazonaws.com` |

### dump Commands

Work with Discogs data dump files.

#### `dump list`

Lists available files in the Discogs data dumps S3 bucket.

| Flag | Description |
|------|-------------|
| `--year` | Filter by year |
| `--month` | Filter by month |
| `--type` | Filter by data type (artists, releases, masters, labels) |
| `--no-table` | Output filenames only (no table formatting) |

#### `dump download <name>`

Downloads a specific data dump file.

| Flag | Description | Default |
|------|-------------|---------|
| `--out-dir` | Output directory | `.` |
| `--overwrite` | Force download even if file exists | `false` |
| `--checksum` | Verify SHA256 checksum after download | `true` |

#### `dump structure <file>`

Analyzes and displays the XML structure of a dump file in a tree format.

| Flag | Description | Default |
|------|-------------|---------|
| `--stop-after` | Stop analysis after X elements | `10000000` |

#### `dump convert <name>`

Converts a dump file to a different format.

| Flag | Description | Default |
|------|-------------|---------|
| `--format` | Output format (`parquet` or `ndjson`) | `parquet` |
| `--out` | Output file path (required for parquet) | - |
| `--no-progress` | Disable progress indicator | `false` |
| `--stop-after` | Stop conversion after X records | `0` (unlimited) |

### db Commands

Work with a PostgreSQL database.

| Flag | Description | Default |
|------|-------------|---------|
| `--database-url` | PostgreSQL connection URL | `postgres://$USER@localhost:5432/dgtools` |

The database URL can also be set via the `DATABASE_URL` environment variable.

#### `db prepare`

Runs database migrations to create the required schema.

#### `db import <file>`

Imports data from a dump file into the database. Automatically truncates relevant tables before import.

#### `db nuke`

Rolls back all migrations, effectively dropping all tables.

---

## Key Strengths

1. **Unified Workflow**: Combines listing, downloading, converting, and importing into a single tool
2. **Checksum Verification**: Automatic SHA256 verification ensures download integrity
3. **Multiple Output Formats**: Supports Parquet (columnar storage) and NDJSON (newline-delimited JSON)
4. **Efficient Database Loading**: Uses PostgreSQL's COPY protocol for high-performance bulk inserts
5. **Single-Pass Parsing**: Parses XML once and distributes records to multiple tables concurrently
6. **Smart Compression**: Uses zstd and dictionary compression in Parquet output
7. **Clean Data Handling**: Converts empty strings to NULL values for cleaner data
8. **Streaming Architecture**: Handles large files without loading them entirely into memory
9. **Schema Migrations**: Uses goose for versioned, reversible database schema changes
10. **Type-Safe Data Models**: Go structs with tags for XML, JSON, and Parquet serialization

---

## Known Limitations

1. **PostgreSQL Only**: Database import only supports PostgreSQL; no support for MySQL, SQLite, or other databases
2. **No Windows Binaries**: Official builds only support Linux and macOS
3. **System Dependency**: Checksum verification requires `sha256sum` command in PATH
4. **No Incremental Updates**: Database imports truncate tables and reload all data
5. **No Data Validation**: Limited validation of imported data beyond what the XML parser provides
6. **Fixed Schema**: Database schema is predetermined; no customization options
7. **Memory for Large Records**: While streaming, individual records must fit in memory

---

## Performance Claims

The repository makes no explicit performance claims or benchmarks. However, the architecture is designed for efficiency:

- Uses channel-based distribution with a 1000-record buffer for backpressure control
- Employs concurrent database writes using connection pooling
- Uses PostgreSQL's COPY protocol which is significantly faster than INSERT statements
- Single-pass XML parsing avoids multiple reads of large files

---

## Discogs Data Dump Parsing

### File Opening (`internal/discogs/dump.go`)

The `OpenDumpFile` function handles both plain XML and gzip-compressed XML files:

```go
func OpenDumpFile(filename string) (*Dump, error) {
    file, err := os.Open(filename)
    // ...
    if !strings.HasSuffix(filename, ".gz") {
        return dd, nil
    }
    gz, err := gzip.NewReader(file)
    // ...
    dd.reader = gz
    dd.Decoder = xml.NewDecoder(dd.reader)
    return dd, nil
}
```

Key characteristics:
- Automatic gzip detection based on `.gz` file extension
- Uses Go's `compress/gzip` for decompression
- Creates an XML decoder for streaming token-based parsing

### XML Parsing (`internal/discogs/dump.go`)

The `DecodeNextElement` method parses XML elements one at a time:

```go
func (dd *Dump) DecodeNextElement() (any, error) {
    t, err := dd.Decoder.Token()
    // ...
    switch se := t.(type) {
    case xml.StartElement:
        switch se.Name.Local {
        case "artist":
            artist := &Artist{}
            dd.Decoder.DecodeElement(artist, &se)
            return artist, nil
        case "label": // ...
        case "master": // ...
        case "release": // ...
        }
    }
    return nil, nil
}
```

Key characteristics:
- Token-based streaming avoids loading entire file into memory
- Recognizes four entity types: artist, label, master, release
- Uses Go's `encoding/xml` package for deserialization
- Returns typed Go structs for each entity

### Filename Parsing (`internal/discogs/dump.go`)

The `DumpFilename` type extracts metadata from standardized Discogs filenames:

```go
var typeExtractor = regexp.MustCompile(`(artists|releases|masters|labels)`)
var dateExtractor = regexp.MustCompile(`discogs_(\d{4})(\d{2})`)

func (fn DumpFilename) Year() string  { return dateExtractor.FindStringSubmatch(string(fn))[1] }
func (fn DumpFilename) Month() string { return dateExtractor.FindStringSubmatch(string(fn))[2] }
func (fn DumpFilename) Type() string  { return typeExtractor.FindStringSubmatch(string(fn))[1] }
```

---

## Data Processing

### Data Cleaning (`internal/discogs/models.go`)

Each model has a `clean()` method that normalizes data:

```go
func (a *Artist) clean() {
    if a.RealName != nil && *a.RealName == "" {
        a.RealName = nil
    }
    if a.Profile != nil && *a.Profile == "" {
        a.Profile = nil
    }
}
```

Key behaviors:
- Converts empty strings to nil/NULL
- Recursively cleans nested objects
- Called during XML unmarshaling

### Record Transformation (`internal/discogs/models.go`)

Models provide `ToRecord()` and related methods for database/file output:

```go
func (a *Artist) ToRecord() []any {
    return []any{
        a.ID,
        a.Name,
        a.RealName,
        a.Profile,
        a.DataQuality,
        a.NameVariations,
        a.URLs,
    }
}
```

---

## Saving Extracted Data

### NDJSON Format

Conversion to NDJSON uses Go's standard `encoding/json`:

```go
case FormatNdjson:
    b, err := json.Marshal(element)
    // ...
    outFile.Write(b)
    outFile.Write([]byte("\n"))
```

Characteristics:
- One JSON object per line
- Uses struct field tags: `json:"field_name"`
- Preserves nested structures (arrays, objects)
- Can output to stdout or file

### Parquet Format

Parquet conversion uses the `parquet-go` library:

```go
case FormatParquet:
    parquetWriter := parquet.NewWriter(outFile)
    // ...
    parquetWriter.Write(element)
    // ...
    parquetWriter.Flush()
```

Model definitions include Parquet-specific tags:

```go
type artist struct {
    ID          int64   `parquet:"id,zstd"`
    Name        string  `parquet:"name,zstd"`
    DataQuality string  `parquet:"data_quality,dict"`
    URLs        []string `parquet:"urls,zstd"`
    // ...
}
```

Compression options:
- `zstd`: Zstandard compression for general fields
- `dict`: Dictionary encoding for low-cardinality fields (e.g., data_quality, status)

---

## Database Loading (PostgreSQL)

### Migration System

Uses the `goose` library with embedded SQL migrations:

```go
//go:embed migrations/pg/*.sql
var embedMigrations embed.FS

func dbPrepare() {
    goose.SetBaseFS(embedMigrations)
    goose.SetDialect("postgres")
    goose.Up(db, "migrations/pg")
}
```

### Single-Pass Import Architecture

The `CopyDiscogsDumpSinglePass` function demonstrates concurrent table loading:

```go
func CopyDiscogsDumpSinglePass(pool *pgxpool.Pool, filename string, modes []int) error {
    channelMap := make(map[int]chan []any)
    bufferSize := 1000

    for _, mode := range modes {
        channelMap[mode] = make(chan []any, bufferSize)
    }

    // Start concurrent writers for each table
    for _, mode := range modes {
        go func(mode int) {
            conn, _ := pool.Acquire(context.Background())
            source := NewCopyFromRecordChannel(channelMap[mode])
            conn.CopyFrom(context.Background(), Tables[mode], getColumnsForMode(mode), source)
        }(mode)
    }

    // Parse XML and distribute to channels
    parser.ParseAndDistribute()
}
```

Key features:
- Single XML parse distributes to multiple table channels
- Buffered channels (1000 records) for flow control
- Concurrent COPY operations to separate tables
- Uses pgx connection pool for efficiency

### COPY Protocol

Uses PostgreSQL's COPY protocol via `pgx.CopyFrom`:

```go
conn.CopyFrom(context.Background(), table, columns, source)
```

The `CopyFromSource` interface is implemented by custom types that feed records from channels or directly from the dump parser.

---

## Data Model

### Entity Types

#### Artist
| Field | Type | PostgreSQL | Notes |
|-------|------|------------|-------|
| id | int64 | integer PK | |
| name | string | varchar | |
| real_name | *string | varchar | |
| profile | *string | text | |
| data_quality | string | varchar | |
| name_variations | []string | jsonb | |
| urls | []string | jsonb | |
| aliases | []*Name | - | Stored in separate table |
| members | []*Name | - | Stored in separate table |

#### Label
| Field | Type | PostgreSQL | Notes |
|-------|------|------------|-------|
| id | int64 | integer PK | |
| name | string | varchar | |
| contact_info | *string | text | |
| profile | *string | text | |
| data_quality | string | varchar | |
| parent_label_id | *int64 | integer | Self-referential |
| urls | []string | jsonb | |

#### Master
| Field | Type | PostgreSQL | Notes |
|-------|------|------------|-------|
| id | int64 | integer PK | From XML attribute |
| title | string | varchar | |
| year | *int32 | integer | |
| main_release_id | *int64 | integer | FK to releases |
| data_quality | string | varchar | |
| genres | []string | jsonb | |
| styles | []string | jsonb | |
| videos | []Video | jsonb | Complex nested type |
| artists | []*MasterArtist | - | Separate join table |

#### Release
| Field | Type | PostgreSQL | Notes |
|-------|------|------------|-------|
| id | int64 | integer PK | From XML attribute |
| status | string | varchar | From XML attribute |
| title | string | varchar | |
| country | *string | varchar | |
| released | *string | varchar | Date as string |
| notes | *string | text | |
| data_quality | string | varchar | |
| master_id | *int64 | integer | FK to masters |
| is_main_release | bool | boolean | |
| genres | []string | jsonb | |
| styles | []string | jsonb | |
| formats | []*ReleaseFormat | jsonb | |
| tracklist | []*Track | jsonb | |
| videos | []*Video | jsonb | |
| companies | []*Company | jsonb | |
| identifiers | []*Identifier | jsonb | |
| series | []*Serie | jsonb | |
| artists | []*MasterArtist | - | Separate join table |
| extra_artists | []*ExtraArtist | - | Separate join table |
| labels | []*ReleaseLabel | - | Separate join table |

### PostgreSQL Database Schema

The schema consists of 10 tables:

**Main Entity Tables:**
- `discogs_artists` - Primary artist records
- `discogs_labels` - Record label information
- `discogs_masters` - Master release groups
- `discogs_releases` - Individual release records

**Join/Relationship Tables:**
- `discogs_artists_aliases` - Artist alias relationships (artist_id, alias_id)
- `discogs_artists_members` - Band membership relationships (artist_id, member_id)
- `discogs_master_artists` - Master-to-artist credits (master_id, artist_id, name, name_variation, join)
- `discogs_release_artists` - Release-to-artist credits
- `discogs_release_extra_artists` - Additional credits (producers, engineers, etc.)
- `discogs_release_labels` - Release-to-label relationships

**Index Strategy:**
- Primary keys on all main entity tables
- Bidirectional indexes on all join tables (A→B and B→A)
- Foreign key indexes for parent references
- Composite index on releases (title, year)

### Parquet Schema

The Parquet output mirrors the Go struct definitions with compression hints:

- **zstd compression**: Applied to most string and numeric fields for good compression
- **dict encoding**: Applied to low-cardinality fields (data_quality, status, country, genre, style)
- **Nested structures**: Arrays and objects are preserved as Parquet nested types

---

## Python Equivalents

For Python developers wanting to replicate dgtools functionality:

| Go Library | Python Equivalent | Purpose |
|------------|-------------------|---------|
| `encoding/xml` | `lxml` or `xml.etree.ElementTree` | XML parsing |
| `compress/gzip` | `gzip` (stdlib) | Gzip decompression |
| `encoding/json` | `json` (stdlib) | JSON serialization |
| `parquet-go/parquet-go` | `pyarrow` or `fastparquet` | Parquet file I/O |
| `jackc/pgx/v5` | `psycopg` or `asyncpg` | PostgreSQL client |
| `pressly/goose/v3` | `alembic` | Database migrations |
| `urfave/cli/v3` | `click` or `typer` | CLI framework |

### Performance Considerations for Python

To achieve similar performance in Python:

1. **Streaming XML**: Use `iterparse` from lxml for memory-efficient parsing
2. **Bulk Database Inserts**: Use `COPY` protocol via `psycopg.copy` or asyncpg's `copy_records_to_table`
3. **Concurrent Processing**: Use `asyncio` with asyncpg or multiprocessing for parallel table loading
4. **Buffered Channels**: Implement with `asyncio.Queue` or `multiprocessing.Queue`

---

## Summary

dgtools is a well-designed, focused tool for working with Discogs data dumps. Its strengths lie in efficient streaming architecture, proper use of PostgreSQL's COPY protocol, and thoughtful output format support. The main limitations are PostgreSQL-only database support and lack of Windows builds.
