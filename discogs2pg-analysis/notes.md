# Research Notes

## Repository Overview
- URL: https://github.com/clrnd/discogs2pg (redirects to github.com/alvare/discogs2pg)
- Author: Ezequiel A. Alvarez
- License: BSD-3-Clause
- Language: Haskell
- Version: 0.3

## Key Files Analyzed

### Configuration Files
- `discogs2pg.cabal`: Package definition, dependencies, build options
- `stack.yaml`: Haskell Stack build configuration (resolver lts-9.17)
- `.travis.yml`: CI/CD pipeline with GitHub Releases deployment

### Source Code Structure
- `src-exe/Main.hs`: Entry point, CLI orchestration
- `src-exe/Options.hs`: Command-line argument parsing with optparse-applicative
- `src/Discogs/Types.hs`: Type exports and Runnable wrapper
- `src/Discogs/Build.hs`: Buildable typeclass for XML parsing
- `src/Discogs/Store.hs`: Table typeclass and PostgreSQL COPY integration
- `src/Discogs/Artist.hs`: Artist entity definition and parsing
- `src/Discogs/Label.hs`: Label entity definition and parsing
- `src/Discogs/Master.hs`: Master release entity definition and parsing
- `src/Discogs/Release.hs`: Release entity (most complex) with nested types
- `src/Discogs/ArtistRelation.hs`: Shared artist relation type

### SQL Schema
- `sql/tables.sql`: Table definitions (15 tables total)
- `sql/indexes.sql`: Primary keys and indexes

## Technical Observations

### XML Parsing
- Uses Hexpat library for fast XML parsing
- `parseThrowing defaultParseOptions` loads entire XML into memory as tree
- Uses lens-simple for nested field manipulation
- Each entity type has `Buildable` instance for parsing

### Data Loading Strategy
- Uses PostgreSQL COPY command (binary protocol)
- Truncates tables before loading (TRUNCATE)
- Opens one connection per table
- Custom escape functions for COPY format
- Arrays serialized as PostgreSQL array literals

### Performance Optimization
- Compiled with `-O2` optimization
- Multi-threaded with `-threaded -rtsopts -with-rtsopts=-N`
- Optional `--aggressive` mode for parallel file processing
- zlib for direct gzip decompression

### Known Data Issues
- Skips artists with empty name field
- Hardcoded skip for release ID 8262262 ("One Vigintillion" with 10^63 tracks)

## Python Equivalent Patterns
- XML parsing: lxml with iterparse for memory efficiency (vs. Hexpat's tree loading)
- Database loading: psycopg2's copy_expert or COPY with StringIO
- Parallelism: multiprocessing or concurrent.futures
- CLI: argparse or click
- Data structures: dataclasses or pydantic models
