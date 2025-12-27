# Analysis: discogs-data-tools

**Repository:** https://github.com/flut1/discogs-data-tools
**Version:** 0.2.0
**Language:** JavaScript (Node.js)
**Author:** Floris Bernard

## Overview

`discogs-data-tools` is a Node.js utility for downloading, verifying, parsing, and importing Discogs monthly data dumps into MongoDB. It provides both a CLI interface and a programmatic Node.js API.

---

## Deployment Options

### 1. Global CLI Installation
```bash
npm install -g discogs-data-tools
discogs-data-tools <command> ...args
```

### 2. Using npx (No Installation Required)
```bash
npx discogs-data-tools <command> ...args
```

### 3. Project Dependency (Node.js API)
```bash
npm install --save discogs-data-tools
```

Then use programmatically:
```javascript
const { dataManager, bucket, fetcher } = require('discogs-data-tools');
```

---

## Installation Requirements

| Requirement | Details |
|-------------|---------|
| **Node.js** | Version 8.2.1 or higher |
| **node-gyp** | Required for native module compilation (`node-expat`). See [node-gyp prerequisites](https://github.com/nodejs/node-gyp#Installation) for platform-specific requirements (Python, C++ build tools) |
| **MongoDB** | Required if using the `mongo` command |

---

## Available Commands

### 1. `fetch` - Download Data Dumps
Downloads gzip-compressed XML data dump files from Discogs S3 bucket.

```bash
discogs-data-tools fetch --latest
discogs-data-tools fetch --target-version 20180101 --collections labels masters
```

**Options:**
- `--interactive, -i` - Interactively select dump version
- `--latest, -l` - Automatically select latest version
- `--target-version, -t` - Specify version (e.g., "20180101")
- `--collections, -c` - Select specific collections: `artists`, `labels`, `masters`, `releases`
- `--data-dir, -d` - Storage directory (default: `./data`)
- `--hide-progress, -n` - Disable progress bar
- `--skip-verify` - Skip checksum verification

### 2. `verify` - Verify Downloaded Dumps
Verifies downloaded dumps using SHA-256 checksums from Discogs.

```bash
discogs-data-tools verify --latest
discogs-data-tools verify --target-version 20180101 --collections releases
```

### 3. `ls` - List Downloaded Data
Lists all downloaded data dump files.

```bash
discogs-data-tools ls --latest
```

### 4. `mongo` - Import to MongoDB
Parses XML dumps and imports data into MongoDB collections.

```bash
discogs-data-tools mongo --latest --connection mongodb://root:pw@127.0.0.1:27017
discogs-data-tools mongo --target-version 20180401 --restart --connection mongodb://root:pw@localhost:27017
```

**Options:**
- `--connection, -o` - MongoDB connection string (required)
- `--database-name, -n` - Database name (default: `discogs`)
- `--chunk-size, -s` - Processing chunk size (default: 1000)
- `--indexes` / `--no-indexes` - Create MongoDB indexes (default: true)
- `--skip-validation` - Skip XML validation for faster processing
- `--drop-existing-collection` - Drop existing collections before import
- `--restart, -r` - Restart from beginning instead of resuming
- `--max-errors, -e` - Maximum errors before abort (default: 100)
- `--bail, -b` - Abort immediately on any error
- `--silent, -m` - Mute console output
- `--include-image-objects` - Include image objects (normally excluded as they lack URIs)

---

## Key Strengths

1. **Streaming XML Parser**
   Uses `node-expat` (C-based libexpat bindings) for efficient memory usage when parsing multi-gigabyte XML files.

2. **Resumable Processing**
   Progress is tracked in `.processing` files, allowing interrupted imports to resume from the last checkpoint.

3. **Data Normalization**
   Automatically transforms inconsistent Discogs data:
   - Converts properties to camelCase
   - Parses numeric strings to integers
   - Splits artist/label names into components (removes `(n)` disambiguation suffixes)
   - Converts track durations to seconds
   - Normalizes release dates to `YYYY` or `YYYY-MM-DD` format

4. **Schema Validation**
   Uses JSON Schema (via AJV) to validate both input XML structures and output documents.

5. **Automatic MongoDB Indexes**
   Creates optimized indexes for common query patterns (text search on names, lookups by ID, filtering by genre/style).

6. **Dual Interface**
   Provides both CLI commands and a Node.js API for programmatic use.

7. **Graceful Exit Handling**
   Cleans up partial downloads if interrupted during fetch.

---

## Known Limitations

1. **MongoDB Only**
   Database import is limited to MongoDB. No support for PostgreSQL, SQLite, or other databases. The readme invites contributions for other databases.

2. **No File Export**
   Does not support exporting processed data to JSON, CSV, or other file formats.

3. **Native Dependencies**
   Requires `node-gyp` and C++ build tools due to `node-expat` dependency, which can be problematic on some systems.

4. **Sequential Collection Processing**
   Collections (artists, labels, masters, releases) are processed one at a time, not in parallel.

5. **Image Data Excluded by Default**
   Image objects are excluded from output because Discogs data dumps don't include image URIs (only dimensions and type).

6. **Node.js API Documentation Incomplete**
   The readme contains a "TODO (coming soon)" placeholder for the custom processing example.

7. **No Incremental Updates**
   Designed for full imports; the `mongo` command overwrites existing documents with the same ID.

---

## Performance Considerations

### Explicit Performance Claims
The readme mentions:
- Using `--skip-validation` "can considerably speed up processing"
- Larger `--chunk-size` "may be more efficient, but costs more memory"

### Performance Mechanisms
1. **Chunk-Based Processing**
   Parser pauses every N records (default: 1000) to process in batches, balancing memory usage with write efficiency.

2. **Bulk Write Operations**
   MongoDB inserts use `bulkWrite` with upsert operations for efficient batch writes.

3. **Streaming with Backpressure**
   XMLParser pauses the underlying stream when processing chunks, preventing memory buildup.

4. **Native XML Parsing**
   `node-expat` is a native C binding to libexpat, significantly faster than pure JavaScript XML parsers.

---

## How It Parses Discogs Data Dump Files

### Parser Implementation (`processing/XMLParser.js`)

The parser is based on [`node-big-xml`](https://github.com/jahewson/node-big-xml) with modifications:

```javascript
class XMLParser extends events.EventEmitter {
  constructor(filename, targetDepth, options) {
    // Uses node-expat for native XML parsing
    this.parser = new expat.Parser('UTF-8');

    // Creates a readable stream from the file
    this.stream = fs.createReadStream(filename);

    // Optionally pipes through gzip decompression
    if (options.gzip) {
      const gunzip = zlib.createGunzip();
      this.stream.pipe(gunzip);
      this.stream = gunzip;
    }
  }
}
```

**Key Design Decisions:**
- **Depth-Based Targeting:** Elements are captured based on XML depth (level 2 for entity records) rather than tag names
- **Event-Driven:** Emits `record` events when complete entities are parsed
- **Pausable:** The `pause()` and `resume()` methods allow backpressure control
- **Gzip Support:** Decompresses `.xml.gz` files on-the-fly using Node.js `zlib`

### Processing Flow (`processing/processor.js`)

```javascript
function processDumpFile(path, collection, fn, gz, chunkSize, restart) {
  // 1. Check for existing progress file
  if (fs.existsSync(progressFilePath) && !restart) {
    toSkip = parseInt(fs.readFileSync(progressFilePath));
  }

  // 2. Create XML parser targeting depth 1 (entity elements)
  const reader = new XMLParser(path, 1, { gzip: gz });

  // 3. Collect records into chunks
  reader.on("record", record => {
    newChunk[chunkIndex] = record;
    if (chunkIndex >= chunkSize) {
      reader.pause();
      fn(oldChunk, collection, path).then(() => {
        // 4. Save progress and resume
        fs.writeFileSync(progressFilePath, processed.toString());
        reader.resume();
      });
    }
  });
}
```

---

## How It Processes Extracted Data

### Data Transformation (`processing/dumpFormatter.js`)

Each entity type has a dedicated formatter function:

| Entity | Function | Output |
|--------|----------|--------|
| Artists | `formatArtist()` | Artist document |
| Labels | `formatLabel()` | Label document |
| Masters | `formatMaster()` | Master document |
| Releases | `formatRelease()` | Release document |

### Transformations Applied

1. **Property Name Conversion**
   - `data_quality` → `dataQuality`
   - `parentLabel` → `parent`
   - `extraartists` → `extraArtists`

2. **Name Parsing** (`parseDiscogsName()`)
   ```javascript
   // Input: "The Beatles (2)"
   // Output: { originalName: "The Beatles (2)", name: "The Beatles", nameIndex: 2 }
   ```

3. **Duration Parsing** (`parseDuration()`)
   ```javascript
   // Input: "3:45"
   // Output: { originalDuration: "3:45", duration: 225 }
   ```

4. **Date Normalization** (`parseReleaseDate()`)
   - `2020/05/15` → `2020-05-15`
   - `202005` → `2020-05-00`
   - Invalid formats are dropped

5. **Filtering**
   - Empty string values excluded
   - Tracks without title or position excluded
   - Identifiers without values excluded
   - Entities without names excluded (with warnings logged)

6. **Image Handling**
   - Image objects excluded by default
   - `imageCount` property set to number of images

---

## How It Saves Extracted Data

### File Storage

The tool stores downloaded data dumps as gzip-compressed XML files:

```
./data/
  └── 20240101/
      ├── discogs_20240101_artists.xml.gz
      ├── discogs_20240101_labels.xml.gz
      ├── discogs_20240101_masters.xml.gz
      ├── discogs_20240101_releases.xml.gz
      └── discogs_20240101_CHECKSUM.txt
```

**Note:** There is no export functionality to save processed data to JSON, CSV, or other formats. The only output destination is MongoDB.

### Processing State Files

During import, progress is tracked:
```
./data/20240101/discogs_20240101_artists.xml.gz.processing
```
Contains the number of records processed for resumability.

---

## How It Loads Data to Databases

### MongoDB (Only Supported Database)

#### Connection

```javascript
const { MongoClient } = require("mongodb");
const client = new MongoClient(argv.connection);
await client.connect();
const db = client.db(argv["database-name"]); // default: "discogs"
```

#### Write Strategy

Uses bulk write with upsert for efficient batch operations:

```javascript
await db.collection(collection).bulkWrite(
  documents.map(({ doc }) => ({
    updateOne: {
      filter: { id: doc.id },
      upsert: true,
      update: doc
    }
  }))
);
```

#### Collections Created

| Collection | Content |
|------------|---------|
| `artists` | Artist entities |
| `labels` | Label/record company entities |
| `masters` | Master release entities |
| `releases` | Individual release entities |

#### Index Creation (`config/mongoIndexSpec.json`)

Indexes are automatically created unless `--no-indexes` is specified:

**Artists Collection:**
- Unique index on `id`
- Unique index on `originalName`
- Text index on `name`, `realName.name`, `nameVariations.name`, `aliases.name`, `groups.name`
- Compound index on `name` + `nameIndex`
- Indexes on `members.id`, `aliases.id`, `groups.id`

**Labels Collection:**
- Unique index on `id`
- Unique index on `originalName`
- Text index on `name`
- Indexes on `sublabels.id`, `parent`

**Masters Collection:**
- Unique index on `id`
- Text index on `title`
- Indexes on `genres`, `styles`, `artists.id`, `artists.name`, `year`

**Releases Collection:**
- Unique index on `id`
- Text index on `title`
- Indexes on `artists.id`, `artists.name`, `companies.id/name`, `labels.id/name/catno`
- Indexes on `genres`, `styles`, `released`
- Compound index on `masterId` + `isMainRelease`

---

## Data Model

### Artist Document

```json
{
  "id": 1234,
  "name": "The Beatles",
  "originalName": "The Beatles",
  "nameIndex": 1,
  "realName": {
    "name": "Real Name",
    "originalName": "Real Name",
    "nameIndex": 1
  },
  "profile": "Biography text...",
  "dataQuality": "Correct",
  "imageCount": 5,
  "urls": ["http://..."],
  "aliases": [
    { "id": 5678, "name": "Alias", "originalName": "Alias", "nameIndex": 1 }
  ],
  "members": [
    { "id": 9012, "name": "John Lennon", "originalName": "John Lennon", "nameIndex": 1 }
  ],
  "groups": [
    { "id": 3456, "name": "Group", "originalName": "Group", "nameIndex": 1 }
  ],
  "nameVariations": [
    { "name": "Beatles", "originalName": "Beatles", "nameIndex": 1 }
  ]
}
```

### Label Document

```json
{
  "id": 1234,
  "name": "Apple Records",
  "originalName": "Apple Records",
  "nameIndex": 1,
  "profile": "Description...",
  "contactInfo": "Contact details...",
  "dataQuality": "Correct",
  "imageCount": 3,
  "urls": ["http://..."],
  "parent": {
    "id": 5678,
    "name": "Parent Label",
    "originalName": "Parent Label",
    "nameIndex": 1
  },
  "sublabels": [
    { "id": 9012, "name": "Sub Label", "originalName": "Sub Label", "nameIndex": 1 }
  ]
}
```

### Master Document

```json
{
  "id": 1234,
  "title": "Abbey Road",
  "year": 1969,
  "mainRelease": 5678,
  "dataQuality": "Correct",
  "notes": "Notes...",
  "imageCount": 10,
  "genres": ["Rock"],
  "styles": ["Pop Rock", "Psychedelic Rock"],
  "artists": [
    {
      "id": 9012,
      "name": "The Beatles",
      "originalName": "The Beatles",
      "nameIndex": 1,
      "join": "&",
      "anv": { "name": "Beatles", "originalName": "Beatles", "nameIndex": 1 }
    }
  ],
  "videos": [
    { "duration": 180, "src": "https://youtube.com/...", "title": "Video Title" }
  ]
}
```

### Release Document

```json
{
  "id": 1234,
  "title": "Abbey Road",
  "released": "1969-09-26",
  "country": "UK",
  "dataQuality": "Correct",
  "notes": "Notes...",
  "masterId": 5678,
  "isMainRelease": true,
  "imageCount": 5,
  "genres": ["Rock"],
  "styles": ["Pop Rock"],
  "artists": [
    {
      "id": 9012,
      "name": "The Beatles",
      "originalName": "The Beatles",
      "nameIndex": 1,
      "role": "Main Artist"
    }
  ],
  "extraArtists": [
    { "id": 3456, "name": "Producer", "role": "Producer" }
  ],
  "labels": [
    { "id": 7890, "name": "Apple Records", "catno": "PCS 7088" }
  ],
  "formats": [
    { "name": "Vinyl", "qty": 1, "descriptions": ["LP", "Album", "Stereo"] }
  ],
  "identifiers": [
    { "type": "Barcode", "value": "5099969944512", "description": "String" }
  ],
  "companies": [
    { "id": 1111, "name": "EMI Studios", "entityType": 10, "entityTypeName": "Recorded At" }
  ],
  "tracklist": [
    {
      "position": "A1",
      "title": "Come Together",
      "duration": 259,
      "originalDuration": "4:19",
      "artists": [...],
      "extraArtists": [...],
      "subTracks": [...]
    }
  ],
  "videos": [
    { "duration": 259, "src": "https://...", "title": "Come Together" }
  ]
}
```

---

## Python Equivalents

For Python developers, equivalent functionality could be achieved with:

| Feature | Node.js Library | Python Equivalent |
|---------|----------------|-------------------|
| XML Parsing | `node-expat` | `lxml.etree.iterparse()` or `xml.sax` |
| MongoDB | `mongodb` | `pymongo` |
| CLI | `yargs` | `argparse` or `click` |
| HTTP Requests | `request` | `requests` or `httpx` |
| Progress Bar | `cli-progress` | `tqdm` |
| JSON Schema | `ajv` | `jsonschema` |
| Checksum | `sumchecker` | `hashlib` (built-in) |
| Gzip | Node.js `zlib` | `gzip` (built-in) |

---

## Summary

`discogs-data-tools` is a focused utility for getting Discogs data dumps into MongoDB. Its strengths lie in efficient streaming XML parsing, automatic data normalization, and resumable processing. The main limitations are MongoDB-only support and lack of file export options. The codebase is well-structured with clear separation between downloading, parsing, formatting, and database operations.
