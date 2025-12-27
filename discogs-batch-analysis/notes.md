# Analysis Notes

## Repository Overview

- **Repository**: https://github.com/echovisionlab/discogs-batch (fork of state303/discogs-batch)
- **Language**: Java 16
- **Build System**: Gradle
- **Framework**: Spring Batch
- **Project Version**: 0.1.8

## Key Files Analyzed

1. `README.md` - Main documentation
2. `build.gradle` - Dependencies and build configuration
3. `src/main/resources/application.yml` - Application configuration
4. Domain classes in `io.dsub.discogs.batch.domain.*`
5. Job configurations in `io.dsub.discogs.batch.job.*`
6. Dump handling in `io.dsub.discogs.batch.dump.*`

## Architecture Observations

### Spring Batch Job Flow
- Each entity type (Artist, Label, Master, Release) has its own step flow
- Step flow: FileFetch -> CoreInsertion -> SubItemsInsertion
- Uses execution deciders to handle conditional step execution

### Multi-threading Strategy
- Uses `ThreadPoolTaskExecutor` with configurable core count
- Default: 80% of physical processor cores
- `SynchronizedItemStreamReader` wraps the StAX readers for thread safety

### XML Parsing
- Uses JAXB (`Jaxb2Marshaller`) for XML-to-object mapping
- Spring OXM's `StaxEventItemReader` for streaming XML parsing
- Handles gzipped XML files directly via `GZIPInputStream`

### Database Operations
- Uses jOOQ for SQL generation and batch operations
- `onConflict().doUpdate()` pattern for upsert operations
- Liquibase for schema management

## Dependencies

External libraries used:
- `spring-boot-starter-batch` - Core batch processing
- `spring-boot-starter-jooq` - jOOQ integration
- `org.springframework:spring-oxm` - OXM for XML binding
- `javax.xml.bind:jaxb-api` - JAXB API
- `io.dsub.discogs:discogs-jooq` - Custom jOOQ schema classes
- `me.tongfei:progressbar` - Terminal progress bars
- `org.liquibase:liquibase-core` - Database migrations
- `com.github.oshi:oshi-core` - System hardware info
- `org.postgresql:postgresql` - PostgreSQL driver

## Entity Relationships Discovered

From analyzing the domain classes:

1. **Artist**
   - Has aliases (artist_alias table)
   - Has groups (artist_group table)
   - Has members (artist_member table)

2. **Label**
   - Has sublabels (label_sub_label table)

3. **Master**
   - Links to main release
   - Has artists (master_artist table)
   - Has videos (master_video table)

4. **Release**
   - Links to master
   - Has artists (release_item_artist table)
   - Has credited artists (release_item_credited_artist table)
   - Has labels (label_release_item table)
   - Has formats (release_item_format table)
   - Has tracks (release_item_track table)
   - Has identifiers (release_item_identifier table)
   - Has works/companies (release_item_work table)
   - Has videos (release_item_video table)

## Data Model Notes

- All entities have `created_at` and `last_modified_at` timestamps (UTC)
- Hash fields used for deduplication in sub-items
- Foreign key relationships maintain referential integrity
- Uses INTEGER for IDs matching Discogs data

## Known Issues from Code

1. `FileFetchTasklet` has a TODO comment: "// TODO: test!"
2. Some test files show malformed data handling scenarios
3. MySQL driver included but only PostgreSQL supported

## Performance Considerations

1. Batch insert with jOOQ's `BatchBindStep`
2. Configurable chunk size (default 3000)
3. Multi-threaded processing with throttle limit
4. Progress bars for visual feedback during long operations
5. Retry logic for deadlock handling (100 retries)
