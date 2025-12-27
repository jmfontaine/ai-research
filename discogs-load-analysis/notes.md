# Research Notes

## Initial Exploration
- Cloned repository to analyze discogs-load
- Project is written in Rust (Cargo.toml workspace structure)
- Main binary: discogs-load
- Has xtask for build/distribution automation

## Project Structure
```
discogs-load/
├── .cargo/config
├── .github/workflows/
│   ├── ci.yml
│   └── release.yml
├── docker/Dockerfile
├── docker-compose.yml
├── discogs-load/           # Main binary crate
│   ├── src/
│   │   ├── main.rs         # Entry point, file handling
│   │   ├── parser.rs       # Parser trait definition
│   │   ├── db.rs           # Database operations
│   │   ├── artist.rs       # Artist entity parser
│   │   ├── label.rs        # Label entity parser
│   │   ├── release.rs      # Release entity parser
│   │   └── master.rs       # Master entity parser
│   └── test_data/          # Sample XML files for testing
├── sql/
│   ├── tables/             # Table creation scripts
│   │   ├── artist.sql
│   │   ├── label.sql
│   │   ├── master.sql
│   │   └── release.sql
│   └── indexes.sql         # Index creation script
└── xtask/                  # Build automation tasks
```

## Key Dependencies
- `quick-xml 0.22.0` - Fast XML parsing
- `flate2 1.0.22` - Gzip decompression
- `postgres 0.19.1` - PostgreSQL client
- `structopt 0.3.17` - CLI argument parsing
- `indicatif 0.16.2` - Progress bars
- `anyhow 1.0` - Error handling
- `log 0.4.0` + `env_logger 0.9.0` - Logging

## Parsing Strategy
1. Opens gzip-compressed XML file
2. First pass: detect file type (labels, releases, artists, masters)
3. Initializes appropriate parser based on root element
4. Second pass: processes all events using state machine pattern
5. Each parser maintains its own state enum for tracking position in XML tree

## State Machine Pattern
Each entity parser implements:
- State enum with states for each XML element
- `process(&mut ev: Event)` method that transitions between states
- Collects data into struct fields as text events are encountered
- On entity end tag, adds to batch HashMap

## Batching Strategy
- Records collected in HashMap keyed by ID
- When batch size reached (default 10,000), writes to database
- After reaching EOF, writes remaining records

## Database Loading
- Uses PostgreSQL COPY BINARY protocol (`COPY ... FROM STDIN BINARY`)
- `BinaryCopyInWriter` for efficient bulk inserts
- Connection created fresh for each batch write
- No transaction management visible (relies on COPY behavior)

## Data Model Analysis
Total: 7 tables across 4 entity types

### artist (1 table)
- id, name, real_name, profile, data_quality
- Arrays: name_variations, urls, aliases, members

### label (1 table)
- id, name, contactinfo, profile, parent_label, data_quality
- Arrays: sublabels, urls

### master (2 tables)
- master: id, title, release_id, year, notes, data_quality + arrays: genres, styles
- master_artist: artist_id, master_id, name, anv, role

### release (3 tables)
- release: id, status, title, country, released, notes, master_id, data_quality + arrays: genres, styles
- release_label: release_id, label_id, label, catno
- release_video: release_id, duration, src, title

## Notable Design Decisions

### Strengths
1. Uses fast Rust + quick-xml for performance
2. Streaming parser avoids loading entire file in memory
3. Binary COPY protocol is fastest way to bulk insert in PostgreSQL
4. Simple state machine makes parsing logic clear
5. Batching prevents memory issues with large datasets
6. Cross-platform builds via GitHub Actions

### Limitations
1. PostgreSQL only - no other databases
2. No file export (CSV, JSON, etc.)
3. Hardcoded progress bar totals (will be wrong for future dumps)
4. No resume/checkpoint capability
5. Re-reads entire file twice (once for type detection)
6. No foreign key constraints defined
7. DROP TABLE before each load (no incremental updates)

## Performance Claims
- README states: "~10 gb compressed and takes ~15 minutes to parse and load on a Mac air m1"
- This is for the releases file, which is the largest

## Inspirations
- Acknowledged in README: discogs-xml2db, discogs2pg
