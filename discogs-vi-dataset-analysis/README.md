# Discogs-VI Dataset Repository Analysis

This document provides a comprehensive analysis of the [MTG/discogs-vi-dataset](https://github.com/MTG/discogs-vi-dataset) GitHub repository.

## Project Overview

Discogs-VI is a dataset of musical version metadata created for research on version identification (VI), also called cover song identification (CSI). It uses editorial metadata from the public Discogs music database to identify version relationships among millions of tracks.

## Deployment Options

The project offers **local deployment only** via Python/Conda environment. There is no Docker, cloud deployment, or containerized options available.

### Local Deployment
```bash
git clone https://github.com/MTG/discogs-vi-dataset
cd discogs-vi-dataset
conda env create -f environment.yaml
conda activate discogs-vi-dataset
```

## Installation Requirements

### System Requirements
- **Operating System**: Linux (tested on)
- **Python Version**: 3.10.9

### Package Manager
- Conda/Miniconda (for environment management)

### Key Dependencies (from `environment.yml`)

| Category | Packages |
|----------|----------|
| XML Processing | `xmltodict` |
| Data Handling | `pandas`, `numpy`, `json` |
| Audio Download | `yt-dlp`, `youtube-dl` |
| Text Processing | `unidecode`, `regex` |
| Web Interface | `streamlit` |
| Unicode | `unicodedata` |

### Data Requirements
- Discogs monthly data dumps (releases and artists XML files)
- ~46 GB for intermediary files (uncompressed)
- ~21 GB for main dataset files (uncompressed)
- ~1.8 TB for downloaded audio (optional)

## Available Commands

### Main Pipeline Script
```bash
./prepare_discogs_vi.sh <release_xml_path> <artist_xml_path> <preprocess_flag>
```
Automates the entire Discogs-VI creation pipeline.

### Individual Processing Commands

| Command | Purpose |
|---------|---------|
| `python discogs_vi/preprocess_releases_xml.py <xml_file>` | Parse releases XML to JSONL |
| `python discogs_vi/preprocess_artists_xml.py <xml_file>` | Parse artists XML to JSONL |
| `python discogs_vi/clean_artists.py <json_file>` | Clean artist metadata |
| `python discogs_vi/clean_releases.py <json_file> <artists_json>` | Clean release metadata |
| `python discogs_vi/parse_releases_to_tracks.py <json_file> <artists_json>` | Convert releases to track format |
| `python discogs_vi/clique_finder.py <tracks_json> <artists_json>` | Find version cliques |

### YouTube Integration Commands

| Command | Purpose |
|---------|---------|
| `python discogs_vi_yt/query_yt/prepare_query_string.py <input_json>` | Prepare YouTube search queries |
| `python discogs_vi_yt/query_yt/query_and_download_yt_metadata.py <queries> <metadata_dir>` | Download YouTube search metadata |
| `python discogs_vi_yt/query_yt/search_tracks_in_queried_yt_metadata.py <input_json> <metadata_dir>` | Match tracks to YouTube videos |
| `python discogs_vi_yt/audio_download_yt/download_missing_version_youtube_urls.py <json_file> <music_dir>` | Download audio files |
| `python discogs_vi_yt/post_processing.py <input_json> <video_dir>` | Post-process and filter dataset |

### Utility Commands

| Command | Purpose |
|---------|---------|
| `python utilities/align_to_official_splits.py <main_path> <videos_path>` | Align downloads to official train/val/test splits |
| `python utilities/prepare_demo.py <jsonl_file>` | Prepare data for Streamlit demo |
| `streamlit run demo.py --server.fileWatcherType -- <demo_json>` | Run interactive demo |
| `utilities/shuffle_and_split.sh <file> <num_splits>` | Split files for parallel processing |

## Key Strengths

1. **Metadata-Based Approach**: Uses writer/composer information for version identification rather than audio similarity, enabling scalability to millions of tracks.

2. **Memory-Efficient XML Parsing**: Uses streaming SAX-style parsing via `xmltodict` with callbacks, processing one record at a time instead of loading entire files.

3. **Rich Data Model**: Captures comprehensive metadata including:
   - Track and release information
   - Artist IDs and names
   - Writer/composer credits
   - Featuring artists
   - Genres and styles
   - YouTube video mappings

4. **Parallel Processing Support**: YouTube metadata download and audio download can be parallelized for efficiency.

5. **Official Benchmark Integration**: Considers Da-TACOS and SHS100K test sets when creating train/validation/test splits for fair comparison.

6. **Sophisticated YouTube Matching**: Multi-stage algorithm verifying:
   - Video category (Music)
   - Duration constraints
   - Officiality (artist channel verification)
   - Title and artist name matching

7. **Data Quality Controls**: Multiple cleaning and filtering stages to handle Discogs data inconsistencies.

8. **Interactive Demo**: Streamlit-based visualization for exploring cliques and versions.

## Known Limitations

1. **Title Matching Constraint**: Two tracks can only be in a clique relationship if they have exactly the same cleaned title (case-insensitive, punctuation-removed).

2. **Artist Count Limit**: Tracks with more than 4 artists or featuring artists are filtered out to simplify YouTube searching.

3. **Writer Disagreements**: When writer annotations conflict across tracks, majority voting is used, potentially losing valid versions with minority annotations.

4. **YouTube Rate Limiting**: Using more than 10 parallel download processes may result in bans. The project recommends 2-10 processes.

5. **Regional Availability**: Audio downloads depend on geographic location; researchers in different regions may get different download success rates.

6. **No Audio Distribution**: The project cannot distribute audio files directly; users must download from YouTube.

7. **Genre Exclusions**: Non-Music and Stage & Screen genres are excluded entirely.

8. **Clique ID Instability**: Clique IDs change between Discogs dump versions (noted as a known issue).

9. **Linux Only**: Tested only on Linux; compatibility with other operating systems is not guaranteed.

## Performance Claims

The documentation provides processing time benchmarks for the July 2024 Discogs dump:

| Operation | Data Size | Time |
|-----------|-----------|------|
| Release XML preprocessing | 17.4M releases | ~3 hours 17 minutes |
| Artist XML preprocessing | 9.2M artists | ~6 minutes |
| Release cleaning | 17.4M → 2.5M releases | ~13 minutes |
| Track parsing | 2.5M releases → 14.4M tracks | ~14 minutes |

### Dataset Scale
- **Discogs-VI**: ~1.9 million versions in ~348,000 cliques
- **Discogs-VI-YT**: ~493,000 versions in ~98,000 cliques (YouTube-matched subset)

## XML Parsing Implementation

The project uses the `xmltodict` library with streaming callbacks for memory-efficient parsing.

### Parsing Flow (discogs_vi/preprocess_releases_xml.py:136-165)
```python
import xmltodict

def get_release(path, release):
    # Callback function for each release element
    # Extracts and transforms data
    # Writes to output file
    return True  # Continue parsing

# Stream parse with callback at depth 2 (release level)
xmltodict.parse(
    open(xml_file, "rb"),
    item_depth=2,
    item_callback=get_release
)
```

### Key Parsing Features
1. **Streaming**: Processes one record at a time (item_depth=2)
2. **Field Simplification**: Normalizes nested structures to lists
3. **Field Removal**: Strips unnecessary metadata (images, notes, companies, etc.)
4. **Type Normalization**: Ensures all repeated fields are lists

### Fields Extracted from Releases
- `id`, `title`, `genres`, `styles`, `artists`, `tracklist`
- `formats`, `labels`, `videos`, `extraartists`
- `country`, `released`, `master_id`

### Fields Extracted from Artists
- `id`, `name`, `aliases`, `groups`
- `namevariations`, `members`

## Data Processing Flow

### Stage 1: XML to JSONL Conversion
- **Input**: Raw Discogs XML dump files
- **Output**: JSONL files with simplified, normalized records
- **Logic**: Stream parse, extract relevant fields, normalize structures

### Stage 2: Artist Cleaning (discogs_vi/clean_artists.py)
- Removes self-references in member/group lists
- Resolves artists with both members and groups (removes members)
- Filters out artists without releases
- Builds name variation ID mappings

### Stage 3: Release Cleaning (discogs_vi/clean_releases.py)
- Filters out releases with excluded genres (Non-Music, Stage & Screen)
- Filters out releases with generic artists (Various, Unknown)
- Extracts writer and featuring artist information
- Removes tracks without writer credits
- Validates genre-style consistency against Discogs taxonomy
- Limits to releases with ≤4 artists

### Stage 4: Track Parsing (discogs_vi/parse_releases_to_tracks.py)
- Converts release-level data to track-level records
- Cleans titles (removes parenthetical content, normalizes text)
- Excludes tracks with generic titles (untitled, intro, outro)
- Associates tracks with release-level metadata

### Stage 5: Clique Finding (discogs_vi/clique_finder.py)
- Groups tracks by cleaned title
- Merges tracks with intersecting writer IDs into versions
- Resolves writer disagreements via majority voting
- Groups versions into cliques based on shared writers
- Filters cliques to require at least 2 versions

### Stage 6: YouTube Matching (discogs_vi_yt/)
1. **Query Preparation**: Creates search strings from track metadata
2. **Metadata Download**: Queries YouTube, saves top 5 results per query
3. **Track Matching**: Compares track metadata with video metadata
4. **Audio Download**: Downloads matched videos as audio (m4a format)
5. **Post-processing**: Filters to downloaded versions, ensures minimum 2 versions per clique

## Data Saving Formats

### JSONL (JSON Lines) - Primary Format

**Encoding**: UTF-8

**Files Using This Format**:
- `discogs_YYYYMMDD_releases.xml.jsonl`
- `discogs_YYYYMMDD_artists.xml.jsonl`
- `Discogs-VI-YYYYMMDD.jsonl`
- `Discogs-VI-YT-YYYYMMDD.jsonl`

**Writing Pattern** (from source):
```python
with open(output_path, "w", encoding="utf-8") as out_f:
    out_f.write(json.dumps(record, ensure_ascii=False) + "\n")
```

**Reading Pattern**:
```python
with open(file_path, encoding="utf-8") as in_f:
    for jsonline in in_f:
        record = json.loads(jsonline)
```

### JSON - Secondary Format

**Encoding**: Default (ASCII)

**Files Using This Format**:
- `Discogs-VI-YT-YYYYMMDD-light.json`
- Train/Val/Test split files

**Writing Pattern**:
```python
with open(output_path, "w") as out_f:
    json.dump(dataset, out_f)
```

**Reading Pattern**:
```python
with open(file_path) as in_f:
    dataset = json.load(in_f)
```

### Text Files

**Format**: Line-delimited plain text (UTF-8)

**Files Using This Format**:
- Query strings: `Discogs-VI-YYYYMMDD.jsonl.queries`
- Lost cliques: `*-lost_cliques.txt`

## Database Loading

**The project does not support loading data to databases.**

All data is stored in file-based formats (JSONL and JSON). While the Conda environment includes SQLAlchemy and SQLite as dependencies, these are not used for dataset storage.

If database storage is required, users would need to implement their own loading scripts using the JSON/JSONL files as input.

## Data Model

### Hierarchical Structure

```
Clique
├── clique_id: string (e.g., "C-0000001")
└── versions: array
    ├── Version 1
    │   ├── version_id: string (e.g., "V-0000001")
    │   └── tracks: array
    │       └── Track (see below)
    └── Version 2
        └── ...
```

### Track Schema

```json
{
  "track_title": "Song Title",
  "release_title": "Album Name",
  "track_writer_ids": ["id1", "id2"],
  "track_writer_names": ["Writer 1", "Writer 2"],
  "track_artist_ids": ["id1"],
  "track_artist_names": ["Artist Name"],
  "track_feat_ids": [],
  "track_feat_names": [],
  "release_id": "12345",
  "release_artist_ids": ["id1"],
  "release_artist_names": ["Release Artist"],
  "release_writer_ids": [],
  "release_writer_names": [],
  "release_feat_ids": [],
  "release_feat_names": [],
  "release_genres": ["Rock"],
  "release_styles": [["Rock", "Classic Rock"]],
  "country": "US",
  "labels": ["Label Name"],
  "formats": ["Vinyl"],
  "master_id": "67890",
  "main_release": "true",
  "release_videos": ["url1", "url2"],
  "released": "1985",
  "track_title_cleaned": "song title"
}
```

### Light Dataset Schema (Discogs-VI-YT-light)

```json
{
  "clique_id": {
    "version_id": "V-0000001",
    "track_title": "Song Title",
    "youtube_id": "dQw4w9WgXcQ"
  }
}
```

### YouTube Video Metadata (in full dataset)

```json
{
  "url": "https://www.youtube.com/watch?v=...",
  "source": "youtube_query",
  "match_type": 0
}
```

Match types range from 0-8, with lower values indicating higher confidence matches.

### Artist Schema (Cleaned)

```json
{
  "id": "12345",
  "name": "Artist Name",
  "aliases": ["id1", "id2"],
  "groups": ["id3"],
  "namevariations": ["Variation 1"],
  "namevariations_id": ["id4", "id5"]
}
```

## Python Equivalents

For projects implementing similar functionality in Python:

| Feature | Library Used | Python Equivalent |
|---------|--------------|-------------------|
| XML Streaming | `xmltodict` | `xml.sax` or `lxml.etree.iterparse` |
| JSON Handling | `json` (stdlib) | Same |
| Unicode Normalization | `unidecode` | Same or `unicodedata` |
| Audio Download | `yt-dlp` | Same |
| Text Cleaning | `re` (stdlib) | Same |
| Web Demo | `streamlit` | Flask, Dash, or FastAPI |

## External Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `xmltodict` | 0.13.0 | XML to dict conversion with streaming |
| `yt-dlp` | 2024.9.27 | YouTube audio downloading |
| `youtube-dl` | 2021.12.17 | Legacy YouTube downloading |
| `streamlit` | 1.30.0 | Interactive web demo |
| `unidecode` | 1.2.0 | Unicode to ASCII transliteration |
| `pandas` | 1.5.3 | Data manipulation |
| `numpy` | 1.23.5 | Numerical operations |

## Conclusion

Discogs-VI-Dataset is a well-structured research tool for creating version identification datasets from Discogs metadata. Its main strengths lie in memory-efficient XML processing, sophisticated matching algorithms, and comprehensive metadata handling. The primary limitation is its metadata-only approach (no audio similarity) and dependence on Discogs data quality. The project outputs file-based JSON/JSONL datasets without database integration.
