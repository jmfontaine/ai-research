# Analysis of discogs-load

This report analyzes the [discogs-load](https://github.com/DylanBartels/discogs-load) GitHub repository, a Rust application for loading Discogs data dumps into PostgreSQL.

## Overview

discogs-load is a command-line tool written in Rust that parses Discogs monthly data dump files (gzip-compressed XML) and loads them into a PostgreSQL database. The project uses a state machine approach with the `quick-xml` library for efficient streaming XML parsing.

---

## Deployment Options

Three deployment methods are available:

### 1. Pre-compiled Binaries (Recommended)
Download pre-built binaries from [GitHub Releases](https://github.com/dylanbartels/discogs-load/releases) for:
- `x86_64-pc-windows-msvc` (Windows x64)
- `aarch64-pc-windows-msvc` (Windows ARM64)
- `x86_64-unknown-linux-gnu` (Linux x64)
- `x86_64-apple-darwin` (macOS x64)
- `aarch64-apple-darwin` (macOS ARM64/Apple Silicon)

```bash
gunzip discogs-load-aarch64-apple-darwin.gz
chmod +x discogs-load-aarch64-apple-darwin
```

### 2. Build from Source
Requires Rust toolchain:
```bash
cargo build --bin discogs-load --release
./target/release/discogs-load --help
```

### 3. Docker Compose
For containerized deployment with PostgreSQL:
```bash
docker-compose up -d postgres
docker-compose up discogs-load
```

---

## Installation Requirements

### For Binary Usage
- Downloaded binary (made executable)
- PostgreSQL database (version 14 recommended based on docker-compose.yml)

### For Building from Source
- Rust toolchain (stable)
- Cargo package manager

The project has no runtime dependencies beyond PostgreSQL connectivity.

---

## Available Commands

### Main Command
```
discogs-load [OPTIONS] [FILE(S)]...
```

### Arguments
| Argument | Description |
|----------|-------------|
| `FILE(S)` | Path to one or more Discogs data dump files (gzip compressed) |

### Options
| Option | Default | Description |
|--------|---------|-------------|
| `--batch-size` | 10000 | Number of rows per insert batch |
| `--db-host` | localhost | PostgreSQL host |
| `--db-name` | discogs | PostgreSQL database name |
| `--db-user` | dev | PostgreSQL username |
| `--db-password` | dev_pass | PostgreSQL password |
| `--create-indexes` | false | Create indexes after loading |
| `-h, --help` | | Print help information |
| `-V, --version` | | Print version |

### Usage Examples
```bash
# Load release and label data
./discogs-load discogs_20211201_releases.xml.gz discogs_20220201_labels.xml.gz

# Create indexes after loading
./discogs-load --create-indexes

# Use custom database settings
./discogs-load --db-host=myhost --db-user=admin file.xml.gz
```

---

## Key Strengths

1. **High Performance**: Written in Rust with the `quick-xml` library for fast XML parsing. Claims ~15 minutes for a ~10 GB compressed file on Apple M1.

2. **Memory Efficient**: Uses streaming parser with batched inserts (default 10,000 records) to avoid loading entire files into memory.

3. **Efficient Database Loading**: Uses PostgreSQL's binary COPY protocol (`COPY ... FROM STDIN BINARY`), the fastest bulk insert method available.

4. **Simple Architecture**: Clean state machine pattern makes the parsing logic easy to understand and maintain.

5. **Cross-Platform**: Pre-built binaries for Windows, Linux, and macOS (both x64 and ARM64).

6. **Minimal Dependencies**: No complex runtime dependencies; just needs PostgreSQL connectivity.

7. **Docker Support**: Includes docker-compose for quick setup with PostgreSQL.

---

## Known Limitations

1. **PostgreSQL Only**: No support for other databases (MySQL, SQLite, etc.).

2. **No File Export**: Cannot export to file formats like CSV, JSON, or Parquet.

3. **Destructive Loading**: Drops and recreates tables on each run; no incremental updates or upserts.

4. **No Resume Capability**: If loading fails mid-process, must restart from the beginning.

5. **Limited Field Extraction**: Does not extract all fields from the Discogs XML (e.g., tracks, images, formats for releases).

6. **Hardcoded Progress Totals**: Progress bar counts are hardcoded and will be inaccurate for future data dumps.

7. **Double File Reading**: Reads each file twice (once to detect type, once to parse).

8. **No Foreign Keys**: Database schema does not define foreign key constraints.

---

## Performance Claims

From the README:
> "At moment of writing the largest file of the monthly dump is ~10 gb compressed and takes ~15 minutes to parse and load on a Mac air m1."

This claim refers to the releases file, which is the largest of the four Discogs dump types.

---

## XML Parsing Mechanism

The project uses a **streaming state machine parser** with the `quick-xml` library:

### Process Flow
1. Opens gzip-compressed file using `flate2::GzDecoder`
2. Wraps in `BufReader` for efficient I/O
3. Creates `quick_xml::Reader` for event-based parsing
4. **First pass**: Reads until root element (`<labels>`, `<releases>`, `<artists>`, or `<masters>`) to determine file type
5. **Second pass**: Processes all XML events through the appropriate parser

### State Machine Pattern (`parser.rs`, `artist.rs`, `label.rs`, `release.rs`, `master.rs`)
Each entity parser:
- Defines a `ParserState` enum with states for each XML element
- Implements `process(&mut self, ev: Event) -> Result<(), Box<dyn Error>>`
- Transitions between states based on `Event::Start`, `Event::End`, and `Event::Text` events
- Accumulates data into struct fields during text events

Example from `artist.rs`:
```rust
enum ParserState {
    Artist, Id, Name, RealName, Profile, DataQuality,
    NameVariations, Url, Urls, Alias, Aliases, Member, Members,
}
```

---

## Data Processing

### Batching Strategy (`artist.rs:127-135`, `label.rs:119-127`, etc.)
1. Records are accumulated in a `HashMap<i32, Entity>` keyed by entity ID
2. When batch reaches configured size (default 10,000), triggers database write
3. HashMap is cleared after each batch write
4. Remaining records written when EOF is reached

### Entity Building
- Current entity is built incrementally as XML events are processed
- On `Event::End` for the entity tag (e.g., `</artist>`), entity is inserted into batch HashMap
- Arrays (e.g., genres, urls) are cleared at `Event::Start` of parent element

---

## Data Saving

### Supported Format
**PostgreSQL database only** - No file export capability.

The project does not support saving to:
- CSV files
- JSON files
- Parquet files
- Other databases

---

## Database Loading

### PostgreSQL Loading Process (`db.rs`)

#### Connection
```rust
// db.rs:187-194
let connection_string = format!(
    "host={} user={} password={} dbname={}",
    db_opts.db_host, db_opts.db_user, db_opts.db_password, db_opts.db_name
);
let client = Client::connect(&connection_string, NoTls)?;
```

#### Table Initialization
- Executes SQL from `sql/tables/*.sql` files
- **Drops existing tables** with `DROP TABLE IF EXISTS ... CASCADE`
- Creates fresh tables for each load

#### Bulk Insert Method
Uses PostgreSQL's binary COPY protocol for maximum performance:

```rust
// db.rs:226-238
fn execute<T>(&self, client: &mut Client, data: &HashMap<i32, T>) -> Result<()>
where T: SqlSerialization
{
    let sink = client.copy_in(&self.copy_stm)?;
    let mut writer = BinaryCopyInWriter::new(sink, self.col_types);
    for values in data.values() {
        writer.write(&values.to_sql())?;
    }
    writer.finish()?;
    Ok(())
}
```

The COPY statement format:
```sql
COPY table_name (columns) FROM STDIN BINARY
```

#### Index Creation
When `--create-indexes` is specified, executes `sql/indexes.sql` after all data is loaded.

---

## Data Model

The destination schema consists of **7 tables** across 4 entity types:

### Artist Table
```sql
CREATE TABLE artist (
    id int NOT NULL,
    name text,
    real_name text,
    profile text,
    data_quality text,
    name_variations text[],
    urls text[],
    aliases text[],
    members text[]
);
```

### Label Table
```sql
CREATE TABLE label (
    id int NOT NULL,
    name text,
    contactinfo text,
    profile text,
    parent_label text,
    sublabels text[],
    urls text[],
    data_quality text
);
```

### Master Tables (2)
```sql
CREATE TABLE master (
    id integer NOT NULL,
    title text,
    release_id integer NOT NULL,
    year integer,
    notes text,
    genres text[],
    styles text[],
    data_quality text
);

CREATE TABLE master_artist (
    artist_id integer NOT NULL,
    master_id integer NOT NULL,
    name text,
    anv text,
    role text
);
```

### Release Tables (3)
```sql
CREATE TABLE release (
    id int NOT NULL,
    status text,
    title text,
    country text,
    released text,
    notes text,
    genres text[],
    styles text[],
    master_id int,
    data_quality text
);

CREATE TABLE release_label (
    id serial,
    release_id int NOT NULL,
    label_id int,
    label text,
    catno text
);

CREATE TABLE release_video (
    id serial,
    release_id int NOT NULL,
    duration int,
    src text,
    title text
);
```

### Indexes (`sql/indexes.sql`)
```sql
ALTER TABLE release ADD CONSTRAINT pkey_release PRIMARY KEY (id);
CREATE INDEX idx_label ON label(id);
CREATE INDEX idx_artist ON artist(id);
CREATE INDEX idx_release ON release(id);
CREATE INDEX idx_release_video ON release_video(release_id);
CREATE INDEX idx_release_label ON release_label(release_id);
CREATE INDEX idx_master_artist_master ON master_artist(master_id);
CREATE INDEX idx_master_artist_artist ON master_artist(artist_id);
```

### Entity Relationship Diagram
The README includes a data model diagram (`imgs/datamodel.png`). Key relationships:
- `release.master_id` → `master.id`
- `release_label.release_id` → `release.id`
- `release_label.label_id` → `label.id`
- `release_video.release_id` → `release.id`
- `master_artist.master_id` → `master.id`
- `master_artist.artist_id` → `artist.id`

Note: These relationships are implied by the data but **not enforced via foreign key constraints** in the schema.

---

## Python Equivalents

For Python developers, the key techniques used map to:

| Rust Component | Python Equivalent |
|---------------|-------------------|
| `quick-xml` event-based parsing | `xml.sax` or `lxml.etree.iterparse()` |
| `flate2::GzDecoder` | `gzip.open()` or `gzip.GzipFile` |
| State machine pattern | Class with state attribute or `enum.Enum` |
| `postgres` crate | `psycopg2` or `psycopg` |
| Binary COPY | `psycopg2.copy_expert()` with binary format |
| `structopt` CLI | `argparse` or `click` |
| HashMap batching | `dict` with periodic flush |

A Python implementation would likely use:
- `lxml.etree.iterparse()` for streaming XML
- `psycopg` with `COPY` for bulk loading
- `dataclasses` for entity definitions

---

## Summary

discogs-load is a focused, high-performance tool for a specific use case: loading Discogs data dumps into PostgreSQL. Its strengths lie in its efficient Rust implementation and use of PostgreSQL's binary COPY protocol. Its limitations are primarily in flexibility—it only supports PostgreSQL and cannot export to files or handle incremental updates.
