# Analysis of discogs-import Repository

**Repository:** https://github.com/bmcfee/discogs-import
**Analysis Date:** 2025-12-27

## Overview

`discogs-import` is a Python tool for importing Discogs data dumps (XML format) into various database backends. It's a fork of the original `discogs-sql-importer` project, extending it with MongoDB and CouchDB support.

---

## Deployment Options

The project supports the following deployment scenarios:

1. **Direct Database Import** - Parse XML and insert directly into PostgreSQL, MongoDB, or CouchDB
2. **JSON Console Output** - Parse XML and output JSON to stdout for piping to other tools
3. **File-based MongoDB Import** - Parse XML to JSON files, then use `mongoimport` for bulk loading

For MongoDB, two URI schemes are supported:
- `mongodb://` - Direct connection and insertion
- `file://` - Write JSON files for later bulk import with `mongoimport`

---

## Installation Requirements

### Python Version
- **Python 2.7** (required - uses Python 2 syntax throughout)

### Core Dependencies (Standard Library)
- `xml.sax` - XML SAX parsing
- `argparse` - Command-line argument parsing
- `json` - JSON serialization

### Optional Dependencies (pip install)
| Package | Purpose |
|---------|---------|
| `ujson` | Faster JSON serialization (optional) |
| `couchdb` | CouchDB backend support |
| `pymongo` | MongoDB backend support |
| `psycopg2` | PostgreSQL backend support |

### Installation
```bash
python setup.py install
```

Or install with specific backend support:
```bash
pip install discogs-import[mongodb]   # For MongoDB
pip install discogs-import[postgres]  # For PostgreSQL
pip install discogs-import[couchdb]   # For CouchDB
```

---

## Available Commands

### Main Parser Script

**`parse_discogs.py`** - Primary command for parsing and importing Discogs data

| Option | Description |
|--------|-------------|
| `-d DATE` / `--date DATE` | Parse all files for a monthly dump (format: YYYYMMDD) |
| `-n N` | Limit parsing to N records per file |
| `-o OUTPUT` / `--output OUTPUT` | Output format: `json`, `pgsql`, `pgdump`, `couch`, `mongo` |
| `-p PARAMS` / `--params PARAMS` | Connection string or output parameters |
| `-i` / `--ignore-unknown-tags` | Continue parsing when encountering unknown XML elements |
| `-q QUALITY` / `--quality QUALITY` | Filter by data quality values |
| `file [file ...]` | Specific XML files to parse |

### Utility Scripts

| Script | Purpose |
|--------|---------|
| `discogs-import-fix-xml.py` | Fixes malformed XML by adding root tags and removing control characters |
| `discogs-import-get_latest_dumps.sh` | Downloads the latest 4 Discogs dump files |
| `discogs-import-make_md5.py` | Generates MD5 checksums for differential imports |
| `discogs-import-make_artist_name_id_pair.py` | Extracts artist name/ID pairs from JSON files |

---

## Key Strengths

1. **Multiple Database Support** - PostgreSQL, MongoDB, and CouchDB backends
2. **Streaming SAX Parser** - Memory-efficient XML processing for large files
3. **Differential Import for MongoDB** - MD5 hashing tracks changes between monthly dumps
4. **Data Quality Filtering** - Import only records meeting specified quality criteria
5. **Flexible Input** - Parse by date convention or specific files
6. **Bulk Import Option** - File-based JSON output enables faster MongoDB imports
7. **Optional Fast JSON** - Uses `ujson` when available for improved performance
8. **Comprehensive Schema** - PostgreSQL schema with proper relationships and constraints

---

## Known Limitations

1. **Python 2.7 Only** - Does not support Python 3.x
2. **No MySQL Support** - Only PostgreSQL among relational databases
3. **No PostgreSQL Upsert** - Direct insert only, no update support for existing records
4. **No Batch Inserts** - PostgreSQL exports one INSERT statement per record (slow)
5. **Name-based References** - PostgreSQL uses artist names instead of IDs for foreign keys
6. **Incomplete XML Support** - Some elements like `videos` and `identifiers` are not parsed
7. **XML Preprocessing Required** - Dump files often need `fix-xml.py` before parsing
8. **Direct MongoDB Import is Slow** - README warns initial import can take days

---

## Performance Claims

From the README documentation:

- **Direct MongoDB import**: "Not overly quick. You might find yourself running the initial import for days."
- **File-based JSON + mongoimport**: "Considerably faster" than direct import
- **Differential imports**: Dramatically reduce data size
  - November 2011 artists: 554MB (2,149,473 records)
  - December 2011 artists (diff): 18MB (48,852 changed records)

---

## XML Parsing Implementation

The project uses Python's built-in SAX (Simple API for XML) parser for streaming XML processing.

### Parser Architecture

```
parse_discogs.py
    |
    +-- xml.sax.make_parser()
    |
    +-- Entity Handlers (ContentHandler subclasses)
        |-- ArtistHandler (artist.py)
        |-- LabelHandler (label.py)
        |-- ReleaseHandler (release.py)
        |-- MasterHandler (master.py)
```

### Parsing Flow

1. **Element Tracking**: Each handler maintains an `inElement` dictionary to track nested XML element state
2. **Character Buffering**: Text content is accumulated in a `buffer` string
3. **Object Construction**: On element close, buffered data is assigned to model object properties
4. **Element Stack**: Release and Master handlers use a stack to track nested context
5. **Export Trigger**: Complete objects are sent to the exporter when closing the root entity element

### Example Handler Logic (simplified)

```python
class ArtistHandler(xml.sax.handler.ContentHandler):
    def startElement(self, name, attrs):
        self.inElement[name] = True
        if name == "artist":
            self.artist = model.Artist()
        elif name == "image":
            # Extract attributes directly
            image = model.ImageInfo()
            image.uri = attrs["uri"]
            self.artist.images.append(image)

    def characters(self, data):
        self.buffer += data

    def endElement(self, name):
        self.buffer = self.buffer.strip()
        if name == 'name':
            if self.inElement['aliases']:
                self.artist.aliases.append(self.buffer)
            else:
                self.artist.name = self.buffer
        elif name == "artist":
            self.exporter.storeArtist(self.artist)
        self.buffer = ''
```

---

## Data Processing

### Transformation Pipeline

1. **XML Parsing** - SAX events converted to Python model objects
2. **Data Quality Check** - Optional filtering based on `data_quality` field
3. **Name Normalization** - MongoDB adds lowercase versions (`l_name`, `l_title`, `l_artist`)
4. **JSON Serialization** - Objects converted to JSON using `__dict__` serialization
5. **Deduplication** - MongoDB supports MD5-based change detection
6. **Export** - Data sent to configured backend

### Data Quality Values

Records can be filtered by these Discogs quality ratings:
- `Needs Vote`
- `Complete And Correct`
- `Correct`
- `Needs Minor Changes`
- `Needs Major Changes`
- `Entirely Incorrect`
- `Entirely Incorrect Edit`

---

## Data Output Formats

### 1. JSON Console Output (`-o json`)

- Serializes Python model objects to JSON using `__dict__`
- Prints one JSON object per line to stdout
- Adds `object_type_name` field indicating the entity type
- Uses `ujson` for serialization if available

### 2. PostgreSQL Dump (`-o pgdump`)

- Generates SQL INSERT statements to stdout
- Same logic as PostgreSQL export but prints instead of executing
- Useful for manual review or batch loading

### 3. MongoDB File Output (`-o mongo -p "file://path/"`)

- Creates separate JSON files per entity type:
  - `artists.json`
  - `labels.json`
  - `releases.json`
  - `masters.json`
- One JSON object per line (JSONL format)
- With `?uniq=md5` option, also creates `.md5` files for tracking changes
- Files can be imported using `mongoimport`:
  ```bash
  mongoimport -d discogs -c artists --ignoreBlanks artists.json
  ```

---

## Database Loading

### PostgreSQL (`-o pgsql`)

**Connection:**
```python
psycopg2.connect(connection_string)
```

**Loading Process:**
1. Connects using psycopg2 connection string
2. For each entity, builds dynamic INSERT statement
3. Executes one INSERT per record (no batching)
4. Handles related data in separate tables (images, formats, tracks, etc.)
5. Commits transaction and closes on finish

**Example:**
```bash
parse_discogs.py -o pgsql -p "host=localhost dbname=discogs user=postgres" -d 20111101
```

### MongoDB (`-o mongo`)

**Connection:**
```python
pymongo.Connection(mongo_uri)
```

**Loading Process:**
1. Connects using MongoDB URI
2. Adds normalized lowercase fields for indexing
3. Adds `updated_on` timestamp
4. Uses `update()` with `upsert=True` for insert-or-update behavior
5. Creates indexes on finish:
   - `id` (unique)
   - `l_name` for artists/labels
   - `l_title`, `l_artist` for releases/masters
   - `format.name` for releases

**Example:**
```bash
parse_discogs.py -o mongo -p "mongodb://localhost/discogs?uniq=md5" -d 20111101
```

### CouchDB (`-o couch`)

**Connection:**
```python
couchdb.Server(server_url)
```

**Loading Process:**
1. Connects to CouchDB server
2. Selects database from URL path
3. Converts objects to JSON via dict serialization
4. Saves documents using `db.save()`

**Example:**
```bash
parse_discogs.py -o couch -p "http://127.0.0.1:5984/discogs" -d 20111101
```

---

## Data Model

### Python Model Classes (model.py)

```
Artist
  - id: int
  - name: str
  - realname: str
  - images: [ImageInfo]
  - urls: [str]
  - namevariations: [str]
  - aliases: [str]
  - profile: str
  - members: [str]
  - groups: [str]

Label
  - id: int
  - name: str
  - images: [ImageInfo]
  - contactinfo: str
  - profile: str
  - parentLabel: str
  - urls: [str]
  - sublabels: [str]

Release
  - id: int
  - status: str
  - title: str
  - country: str
  - released: str
  - notes: str
  - genres: [str]
  - styles: [str]
  - images: [ImageInfo]
  - formats: [Format]
  - labels: [ReleaseLabel]
  - artist: str
  - artists: [str]
  - artistJoins: [ArtistJoin]
  - tracklist: [Track]
  - extraartists: [Extraartist]

Master
  - id: int
  - title: str
  - main_release: int
  - year: int
  - notes: str
  - genres: [str]
  - styles: [str]
  - images: [ImageInfo]
  - artist: str
  - artists: [str]
  - artistJoins: [ArtistJoin]
  - extraartists: [Extraartist]

Supporting Types:
  - ImageInfo (height, width, type, uri, uri150)
  - Format (name, qty, descriptions)
  - ReleaseLabel (name, catno)
  - Track (title, duration, position, artists, artistJoins, extraartists)
  - ArtistJoin (artist1, join_relation)
  - Extraartist (name, roles)
```

### PostgreSQL Schema

The schema consists of 21 tables with the following structure:

**Core Entity Tables:**
| Table | Primary Key | Key Columns |
|-------|-------------|-------------|
| `artist` | `id` | name, realname, profile, urls[], aliases[], members[], groups[] |
| `label` | `id` | name, contactinfo, profile, parent_label, sublabels[], urls[] |
| `release` | `id` | title, status, country, released, notes, genres, styles, master_id |
| `master` | `id` | title, main_release, year, notes, genres, styles |

**Supporting Tables:**
| Table | Purpose |
|-------|---------|
| `image` | Shared image storage (PK: uri) |
| `format` | Media format types (PK: name) |
| `genre` | Genre definitions |
| `country` | Country names |
| `role` | Artist roles |
| `track` | Release tracks |

**Junction Tables:**
| Table | Relationship |
|-------|--------------|
| `artists_images` | Artist to Image |
| `labels_images` | Label to Image |
| `releases_images` | Release to Image |
| `masters_images` | Master to Image |
| `releases_artists` | Release to Artist (name-based) |
| `releases_labels` | Release to Label (with catno) |
| `releases_formats` | Release to Format (with qty, descriptions) |
| `releases_extraartists` | Release to Extra Artist (with roles) |
| `releases_artists_joins` | Artist collaboration joins |
| `tracks_artists` | Track to Artist |
| `tracks_extraartists` | Track to Extra Artist |
| `tracks_artists_joins` | Track artist collaboration joins |
| `tracks_extraartists_roles` | Detailed track artist roles |
| `masters_artists` | Master to Artist |
| `masters_extraartists` | Master to Extra Artist |
| `masters_artists_joins` | Master artist collaboration joins |

### MongoDB Collections

MongoDB stores documents in 4 collections:
- `artists` - Artist documents with added `l_name` (lowercase name)
- `labels` - Label documents with added `l_name`
- `releases` - Release documents with added `l_title`, `l_artist`
- `masters` - Master documents with added `l_title`, `l_artist`

**Indexes Created:**
```javascript
// Artists
db.artists.ensureIndex({id: 1}, {unique: true})
db.artists.ensureIndex({l_name: 1})

// Labels
db.labels.ensureIndex({id: 1}, {unique: true})
db.labels.ensureIndex({l_name: 1})

// Releases
db.releases.ensureIndex({id: 1}, {unique: true})
db.releases.ensureIndex({l_artist: 1, l_title: 1})
db.releases.ensureIndex({'format.name': 1})

// Masters
db.masters.ensureIndex({id: 1}, {unique: true})
db.masters.ensureIndex({l_title: 1})
db.masters.ensureIndex({main_release: 1}, {unique: true})
```

---

## Design Patterns and Techniques

### SAX Streaming Parser
- Memory-efficient for multi-gigabyte XML files
- Event-driven processing without loading entire DOM

### Strategy Pattern for Exporters
- Exporters implement common interface (`storeArtist`, `storeLabel`, etc.)
- Selected at runtime based on `-o` option
- Easy to add new backends

### MD5 Change Detection
- Computes hash of JSON representation
- Compares against stored hashes from previous import
- Only processes changed records

### Dynamic SQL Building
- PostgreSQL exporter builds INSERT statements dynamically
- Only includes columns with non-empty values
- Uses parameterized queries for safety

### Equivalent Python Patterns
For modern Python 3 implementations:
- Replace `xml.sax` with `lxml.etree.iterparse()` for better performance
- Use `asyncio` for concurrent database operations
- Replace manual SQL building with SQLAlchemy ORM
- Use `dataclasses` for model definitions
- Consider `orjson` for fastest JSON serialization

---

## Summary

`discogs-import` is a functional but dated tool (Python 2.7) for importing Discogs data dumps. Its main value lies in:

1. **Complete implementation** for three database backends
2. **Streaming architecture** suitable for large files
3. **Differential import** capability for MongoDB
4. **Comprehensive PostgreSQL schema** as a reference design

Key areas for improvement in a modern rewrite would be:
- Python 3 compatibility
- Batch database operations
- Async I/O for better performance
- Proper ORM usage
- Support for additional XML elements (videos, identifiers)
