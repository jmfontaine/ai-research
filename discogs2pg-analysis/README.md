# Analysis: discogs2pg

**Repository:** https://github.com/clrnd/discogs2pg (redirects to alvare/discogs2pg)
**Author:** Ezequiel A. Alvarez
**License:** BSD-3-Clause
**Language:** Haskell
**Version:** 0.3

## Overview

discogs2pg is a command-line tool for importing Discogs XML data dumps into PostgreSQL. Written in Haskell, it focuses on performance by using the Hexpat XML parser and PostgreSQL's COPY command for efficient bulk data loading.

---

## Deployment Options

| Option | Description |
|--------|-------------|
| **Build from source** | Use Haskell Stack to compile locally |
| **GitHub Releases** | Pre-built Linux binaries available via Travis CI automated releases |

The project uses Travis CI for continuous integration and automatically deploys tagged releases to GitHub Releases as pre-built binaries.

---

## Installation Requirements

### Prerequisites
1. **Haskell Stack** - The Haskell build tool (https://github.com/commercialhaskell/stack)
2. **PostgreSQL** - Target database with client tools (`createdb`, `psql`)
3. **libpq** - PostgreSQL client library (for postgresql-simple Haskell package)

### Installation Steps
```bash
# Install Stack if not already installed
# See: https://docs.haskellstack.org/en/stable/install_and_upgrade/

# Clone and build
git clone https://github.com/clrnd/discogs2pg.git
cd discogs2pg
stack install

# Ensure ~/.local/bin is in your PATH
```

### Dependencies (Haskell packages)
| Package | Purpose |
|---------|---------|
| hexpat | Fast XML parsing |
| postgresql-simple | PostgreSQL database connectivity |
| lens-simple | Functional data access patterns |
| optparse-applicative | Command-line argument parsing |
| zlib | Gzip decompression |

---

## Available Commands

The tool provides a single executable `discogs2pg` with the following options:

| Option | Description |
|--------|-------------|
| `-c, --conn CONN` | PostgreSQL libpq connection string (required) |
| `-d, --date DATE` | Process all XML files for a specific date (format: YYYYMMDD) |
| `-g, --gzip` | Read compressed `.xml.gz` files directly |
| `--aggressive` | Process all file types in parallel (opens 15 connections) |
| `FILE` | Process a single XML file |

### Usage Modes

**Single file import:**
```bash
discogs2pg -c "host=localhost dbname=discogs" artists.xml
discogs2pg -c "dbname=discogs" -g releases.xml.gz
```

**Batch import by date:**
```bash
# Imports discogs_20150810_{artists,labels,masters,releases}.xml
discogs2pg -c "dbname=discogs" -d 20150810

# With gzip compression
discogs2pg -c "dbname=discogs" -d 20150810 -g

# Parallel processing (faster but uses 15 connections)
discogs2pg -c "dbname=discogs" -d 20150810 -g --aggressive
```

### Complete Workflow
```bash
createdb discogs
psql discogs < sql/tables.sql
discogs2pg -g -d 20150810 -c "dbname=discogs"
psql discogs < sql/indexes.sql
```

---

## Key Strengths

1. **Performance-focused design**
   - Uses Hexpat, a fast C-based XML parser with Haskell bindings
   - PostgreSQL COPY command for high-speed bulk inserts (bypasses transaction overhead)
   - Multi-threaded execution with GHC runtime (`-threaded -with-rtsopts=-N`)
   - Optional parallel file processing with `--aggressive` mode

2. **Direct gzip support**
   - Decompresses files in-memory without temporary files
   - Uses zlib for efficient decompression

3. **Comprehensive data model**
   - 15 tables covering artists, labels, masters, releases, and all related entities
   - Preserves relationships (artist credits, track artists, companies, etc.)
   - Uses PostgreSQL arrays for multi-valued attributes

4. **Type-safe implementation**
   - Haskell's strong type system prevents data corruption
   - Lens-based data access ensures consistent field handling

5. **Clean separation of concerns**
   - Distinct modules for parsing (Build), storage (Store), and entity types
   - Typeclass-based polymorphism enables consistent handling across entity types

---

## Known Limitations

1. **PostgreSQL only**
   - No support for other databases (SQLite, MySQL, etc.)
   - No file-based export formats (CSV, JSON)

2. **Memory-intensive parsing**
   - Loads entire XML document into memory as a tree
   - May require significant RAM for large dump files

3. **Schema rigidity**
   - Tables are truncated on each import (no incremental updates)
   - Requires manual schema creation before import

4. **Limited error handling**
   - Errors halt the entire import process
   - No resume capability after failures

5. **Hardcoded workarounds**
   - Skips release ID 8262262 (problematic entry with 10^63 tracks)
   - Skips artists with empty name fields

6. **Single PostgreSQL schema**
   - No option to specify target schema
   - Foreign keys are commented out (not enforced)

7. **Outdated Haskell stack**
   - Resolver lts-9.17 uses GHC 8.0.2 (from 2016)

---

## Performance Claims

The README explicitly states performance goals:

1. **Motivation:** The author notes existing solutions were "slow" (5+ hours without completion)
2. **COPY vs INSERT:** Uses PostgreSQL COPY instead of individual INSERTs for efficiency
3. **Parallel processing:** The `--aggressive` mode can process all files simultaneously
4. **Claimed time:** "Wait an hour or two" for a full import (vs. 5+ hours with alternatives)
5. **Commodity hardware target:** Designed to work well on standard machines

**Build optimization flags:**
- `-O2` for maximum compiler optimization
- `-threaded -rtsopts -with-rtsopts=-N` for automatic multi-core utilization

---

## Parsing: How Discogs Data Dumps Are Parsed

### XML Parser
The tool uses **Hexpat** (`Text.XML.Expat.Tree`), which is a Haskell binding to the C Expat XML parser.

### Parsing Pipeline
```
File (gzip optional) → Decompress → Parse XML Tree → Build Entities
```

### Implementation Details

**File Reading (`Main.hs:26-28`):**
```haskell
let read_f = case (isGzip opts) of
                 True -> fmap decompress . LB.readFile
                 False -> LB.readFile
```

**XML Parsing (`Main.hs:40-41`):**
```haskell
parseThrowing defaultParseOptions  -- Parse to UNode ByteString tree
```

**Entity Building (`Discogs/Build.hs`):**
- `Buildable` typeclass defines `build :: UNode ByteString -> [a]`
- Each entity type (Artist, Label, Master, Release) implements `Buildable`
- Uses `foldl'` for strict left fold over XML child nodes
- Lens setters map XML elements to entity fields

**Example Artist Parsing (`Artist.hs:66-81`):**
```haskell
parseArtist' :: UNode ByteString -> Artist -> Artist
parseArtist' (Element "id" _ txt) = artistId .~ getTexts txt
parseArtist' (Element "name" _ txt) = artistName .~ getTexts txt
parseArtist' (Element "aliases" _ ns) = artistAliases .~ getNodes "name" ns
-- ... etc
```

---

## Processing: How Extracted Data Is Processed

### Entity Types
The tool defines Haskell data types for each entity:

| Entity | Fields | Related Tables |
|--------|--------|----------------|
| Artist | id, name, realname, profile, urls, aliases, groups, members, namevariations, data_quality | 1 |
| Label | id, name, contactinfo, profile, parent_label, sublabels, urls, data_quality | 1 |
| Master | id, title, main_release, year, notes, genres, styles, data_quality + artists | 2 |
| Release | 18 fields + nested labels, formats, tracks, identifiers, videos, companies, artists | 11 |

### Processing Flow
1. **Automatic type detection:** Single-file mode detects entity type from root XML element
2. **Streaming construction:** Entities built lazily as XML is traversed
3. **Nested entity flattening:** Releases decompose into 11 separate table outputs
4. **Data validation:** Entries with empty required fields are skipped with warnings

### Data Transformations
- **Arrays:** Multi-valued fields become PostgreSQL arrays (e.g., `genres text[]`)
- **Relations:** Nested artist references extracted to join tables
- **Track indexing:** Tracks receive sequential index numbers
- **Escaping:** Special characters escaped for COPY format (tabs, newlines, backslashes)

---

## Saving Data: Output Formats

### Supported Format: PostgreSQL Only

discogs2pg does **not** support file-based output formats (CSV, JSON, etc.). Data flows directly from XML parsing to PostgreSQL via the COPY protocol.

The tool is specifically designed for PostgreSQL loading and does not include any intermediate file persistence.

---

## Loading Data to Databases

### Supported Database: PostgreSQL Only

The tool exclusively supports PostgreSQL using the `postgresql-simple` library.

### Loading Mechanism

**PostgreSQL COPY Protocol (`Store.hs:39-66`):**

1. **Connection setup:**
   - One connection per table
   - Begins transaction with `BEGIN`
   - Truncates existing data with `TRUNCATE table_name`

2. **COPY initiation:**
   ```haskell
   copy_ conn "COPY table_name (columns...) FROM STDIN"
   ```

3. **Data streaming:**
   - Entities converted to tab-separated rows via `toRows`
   - Rows sent with `putCopyData`
   - Uses ByteString Builder for efficient serialization

4. **Completion:**
   - `putCopyEnd` returns row count
   - `COMMIT` finalizes transaction

### Escaping Logic (`Store.hs:75-122`)

The `Escapable` typeclass handles PostgreSQL COPY format requirements:

| Character | Escape Sequence |
|-----------|-----------------|
| `\n` (newline) | `\n` |
| `\t` (tab) | `\t` |
| `\r` (carriage return) | `\r` |
| `\` (backslash) | removed (stripped) |
| Empty string | `\N` (NULL) |
| Arrays | `{\"val1\",\"val2\"}` |

### Connection String

Uses libpq connection string format:
```bash
-c "host=localhost port=5432 dbname=discogs user=postgres"
-c "dbname=discogs"  # Simplified for local connections
```

---

## Data Model

### Entity-Relationship Overview

```
                    ┌─────────────┐
                    │   artist    │
                    └──────┬──────┘
                           │
         ┌─────────────────┼─────────────────┐
         ▼                 ▼                 ▼
┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐
│ release_artist  │ │master_artist│ │  track_artist   │
└────────┬────────┘ └──────┬──────┘ └────────┬────────┘
         │                 │                 │
         ▼                 ▼                 ▼
    ┌─────────┐       ┌─────────┐       ┌─────────┐
    │ release │◄──────│ master  │       │  track  │
    └────┬────┘       └─────────┘       └────┬────┘
         │                                   │
         └───────────────────────────────────┘
         │
         ├──► release_label
         ├──► release_format
         ├──► release_identifier
         ├──► release_video
         ├──► release_company
         ├──► release_extraartist
         └──► track_extraartist
```

### Table Definitions

#### Core Entities

**artist**
| Column | Type | Description |
|--------|------|-------------|
| id | integer | Primary key |
| name | text | Artist name |
| realname | text | Real name |
| urls | text[] | Website URLs |
| namevariations | text[] | Name variations |
| aliases | text[] | Alias names |
| releases | integer[] | Associated release IDs |
| profile | text | Biography |
| members | text[] | Group members |
| groups | text[] | Groups artist belongs to |
| data_quality | text | Data quality rating |

**label**
| Column | Type | Description |
|--------|------|-------------|
| id | integer | Primary key |
| name | text | Label name |
| contactinfo | text | Contact information |
| profile | text | Label profile |
| parent_label | text | Parent label name |
| sublabels | text[] | Sub-label names |
| urls | text[] | Website URLs |
| data_quality | text | Data quality rating |

**master**
| Column | Type | Description |
|--------|------|-------------|
| id | integer | Primary key |
| title | text | Master release title |
| main_release | integer | Main release ID |
| year | integer | Release year |
| notes | text | Notes |
| genres | text[] | Genre tags |
| styles | text[] | Style tags |
| data_quality | text | Data quality rating |

**release**
| Column | Type | Description |
|--------|------|-------------|
| id | integer | Primary key |
| status | text | Release status |
| title | text | Release title |
| country | text | Country of release |
| released | text | Release date |
| notes | text | Notes |
| genres | text[] | Genre tags |
| styles | text[] | Style tags |
| master_id | integer | Master release reference |
| data_quality | text | Data quality rating |

#### Relationship Tables

**master_artist / release_artist / release_extraartist**
| Column | Type | Description |
|--------|------|-------------|
| master_id/release_id | integer | Parent entity ID |
| artist_id | integer | Artist ID |
| anv | text | Artist name variation used |
| join_relation | text | Join phrase (e.g., "feat.") |
| role | text | Artist role/credit |

**track**
| Column | Type | Description |
|--------|------|-------------|
| release_id | integer | Release ID |
| idx | integer | Track sequence number |
| position | text | Position label (e.g., "A1") |
| title | text | Track title |
| duration | text | Duration string |

**track_artist / track_extraartist**
| Column | Type | Description |
|--------|------|-------------|
| track_idx | text | Track index |
| release_id | integer | Release ID |
| artist_id | integer | Artist ID |
| anv | text | Artist name variation |
| join_relation | text | Join phrase |
| role | text | Artist role |

**release_format**
| Column | Type | Description |
|--------|------|-------------|
| release_id | integer | Release ID |
| format_name | text | Format (e.g., "Vinyl") |
| format_text | text | Format text |
| qty | bigint | Quantity |
| descriptions | text[] | Format descriptions |

**release_label**
| Column | Type | Description |
|--------|------|-------------|
| release_id | integer | Release ID |
| label | text | Label name |
| catno | text | Catalog number |

**release_identifier**
| Column | Type | Description |
|--------|------|-------------|
| release_id | integer | Release ID |
| description | text | Identifier description |
| type | text | Identifier type |
| value | text | Identifier value |

**release_video**
| Column | Type | Description |
|--------|------|-------------|
| release_id | integer | Release ID |
| duration | integer | Video duration |
| src | text | Video URL |
| title | text | Video title |

**release_company**
| Column | Type | Description |
|--------|------|-------------|
| release_id | integer | Release ID |
| company_id | integer | Company ID |
| entity_type | integer | Entity type code |
| entity_type_name | text | Entity type name |
| catno | text | Catalog number |

### Indexes

Primary keys are defined for:
- artist(id)
- label(id)
- master(id)
- release(id)
- track(release_id, idx)

Secondary indexes for common query patterns:
- master_artist(master_id), master_artist(artist_id)
- release(title), release(country)
- release_artist(artist_id), release_artist(release_id)
- release_extraartist(artist_id), release_extraartist(release_id)
- track_artist(artist_id), track_artist(release_id), track_artist(track_idx)
- release_label(release_id), release_label(label), release_label(catno)
- label(name), artist(name)

**Note:** Foreign key constraints are defined but commented out in the schema, likely for import performance.

---

## Python Equivalent Patterns

For Python developers, here are equivalent approaches:

| Haskell Approach | Python Equivalent |
|------------------|-------------------|
| Hexpat tree parsing | `lxml.etree.parse()` or `xml.etree.ElementTree` |
| Streaming for memory | `lxml.etree.iterparse()` with element clearing |
| PostgreSQL COPY | `psycopg2.copy_expert()` with StringIO |
| Lens-based data access | `dataclasses` with `__slots__` or `pydantic` models |
| optparse-applicative | `argparse` or `click` |
| Parallel processing | `concurrent.futures.ProcessPoolExecutor` |
| zlib decompression | `gzip.open()` or `gzip.decompress()` |
| ByteString Builder | `io.BytesIO` with buffered writing |
