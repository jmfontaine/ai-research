# Analysis Notes

## Repository Overview
- **Project Name**: Discogs-VI Dataset
- **Purpose**: Create musical version identification (VI) / cover song identification (CSI) datasets
- **License**: AGPL-3.0 (code), CC BY-NC-SA 4.0 (metadata)
- **Language**: Python 3.10.9
- **Primary Author**: MTG (Music Technology Group)

## Key Findings

### XML Parsing Approach
- Uses `xmltodict` library with streaming callback (`item_depth=2`)
- Memory efficient: processes one record at a time
- Converts XML to JSON Line format for easier processing

### Data Processing Pipeline
1. `preprocess_releases_xml.py` - Parse releases XML to JSONL
2. `preprocess_artists_xml.py` - Parse artists XML to JSONL
3. `clean_artists.py` - Fix artist relationships and IDs
4. `clean_releases.py` - Filter and clean release metadata
5. `parse_releases_to_tracks.py` - Convert releases to track format
6. `clique_finder.py` - Group versions into cliques based on writer matching

### YouTube Integration
- Query preparation using track metadata
- Uses `yt-dlp` for downloading audio
- Sophisticated matching algorithm checking:
  - Video category (must be "Music")
  - Duration limit (max 20 minutes)
  - Officiality verification
  - Title and artist matching

### Output Formats
- **JSONL**: Primary format for large datasets (UTF-8 encoded)
- **JSON**: Used for smaller datasets and light versions (default encoding)
- **Text files**: For query strings

### No Database Support
- Project does not load to any databases
- All data stored as file-based JSON/JSONL

### Performance Notes
- Release preprocessing takes ~3 hours for 17M releases
- Artist preprocessing takes ~6 minutes for 9M artists
- Release cleaning takes ~13 minutes
- Track parsing takes ~14 minutes
- Clique finding time depends on data size

### Data Model
- Hierarchical: Clique → Versions → Tracks
- Each track contains rich metadata (artist IDs/names, writer info, genres, styles)
- UUIDs for clique and version identification

## Interesting Design Decisions
1. Uses writer matching for version identification (not audio similarity)
2. Streaming XML parser for memory efficiency
3. Multiple cleaning stages to handle Discogs data inconsistencies
4. Majority voting for conflicting writer annotations
5. Supports parallel YouTube downloading with rate limiting warnings
