# Analysis Notes

## Repository Overview
- **URL:** https://github.com/flut1/discogs-data-tools
- **Language:** JavaScript (Node.js)
- **Version:** 0.2.0
- **Author:** Floris Bernard

## Key Files Analyzed
1. `readme.md` - Main documentation
2. `package.json` - Dependencies and project metadata
3. `api.md` - Node.js API documentation
4. `cli/index.js` - CLI command definitions
5. `processing/XMLParser.js` - XML parsing logic
6. `processing/processor.js` - Chunk-based processing
7. `processing/dumpFormatter.js` - Data transformation logic
8. `cli/mongo.js` - MongoDB import logic
9. `config/mongoIndexSpec.json` - MongoDB index definitions
10. `schema/docs/*.json` - JSON schemas for output documents
11. `util/parseUtils.js` - Parsing utilities

## Architecture Observations

### XML Parsing Strategy
- Uses `node-expat` library (based on expat C library)
- Custom XMLParser class extends EventEmitter
- Streaming approach with pause/resume capability
- Targets elements at specific depth rather than by name
- Supports gzip-compressed files via Node.js zlib

### Data Processing Pipeline
1. Read gzip-compressed XML dump from disk
2. Stream-parse XML using node-expat
3. Collect records into chunks (default 1000)
4. Pause parser, process chunk
5. Resume parser for next chunk
6. Track progress in `.processing` file for resumability

### Data Transformation
- All properties converted to camelCase
- Numeric strings parsed to integers
- Artist/label names split into: originalName, name, nameIndex
- Duration converted to seconds
- Images replaced with imageCount (no URIs in dumps)
- Empty values filtered out
- Invalid dates normalized or dropped

### MongoDB Integration
- Uses official MongoDB Node.js driver
- Bulk write operations with upsert
- Automatic index creation
- Collection per entity type (artists, labels, masters, releases)
- Schema validation using AJV

## External Dependencies
- `node-expat` - Fast C-based XML parsing
- `mongodb` - MongoDB driver
- `ajv` - JSON Schema validation
- `yargs` - CLI argument parsing
- `inquirer` - Interactive prompts
- `cli-progress` - Progress bar
- `request` / `request-progress` - HTTP downloads
- `sumchecker` - Checksum verification
- `xml2js` - S3 bucket listing parsing
- `lodash` - Utility functions
- `fs-extra` - Enhanced file system operations

## Limitations Discovered
- MongoDB-only database support
- No export to file formats (JSON, CSV, etc.)
- Requires node-gyp for native module compilation
- No parallel processing of collections
- Progress tracking only works with uninterrupted runs
