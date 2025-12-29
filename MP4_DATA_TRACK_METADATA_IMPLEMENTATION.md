# MP4 Data Track Metadata Implementation for hls.js

## Overview

This document describes the implementation that extends hls.js to parse metadata from MP4 data tracks (timed metadata tracks) in HLS fragments. This complements the existing CMAF EMSG metadata support.

## Key Differences: EMSG vs Data Track

| Aspect           | EMSG (Existing)               | Data Track (New)                          |
| ---------------- | ----------------------------- | ----------------------------------------- |
| **Location**     | Fragment-level boxes (`emsg`) | Track-level samples in `mdat`             |
| **Parsing**      | `findBox()` + `parseEmsg()`   | Parse `traf`/`trun` + extract from `mdat` |
| **Timing**       | From emsg `presentationTime`  | From `tfdt` + `trun` durations            |
| **Init segment** | Not needed                    | Must parse to find track ID               |
| **Schema ID**    | `schemeIdUri` string          | fourCC in `stsd`                          |

## Files Modified

### 1. Configuration (`src/config.ts`)

**Lines: 253-261, 462-463**

Added two new configuration options to `MetadataControllerConfig`:

```typescript
export type MetadataControllerConfig = {
  // ... existing options ...
  enableDataTrackMetadata: boolean;
  dataTrackMetadataTrackId?: number;
};
```

**Default values:**

- `enableDataTrackMetadata: false` - Enable extraction of metadata from MP4 data tracks
- `dataTrackMetadataTrackId: undefined` - Auto-detect metadata track if undefined

**Note:** Data track metadata is delivered only via the `FRAG_PARSING_METADATA` event, not as VTTCues on the ID3 text track.

### 2. MP4 Tools (`src/utils/mp4-tools.ts`)

**Lines: 1059-1196**

Added new exported function `parseDataTrackSamples()`:

**Purpose:** Extracts metadata samples from MP4 data tracks (timed metadata tracks)

**Parameters:**

- `data: Uint8Array` - MP4 fragment data containing moof+mdat boxes
- `initData: InitData` - Parsed init segment data with track information
- `metaTrackId: number` - Track ID of the metadata track
- `timeOffset: number` - Time offset to apply to sample timestamps

**Returns:** `MetadataSample[]` - Array of metadata samples

**Implementation details:**

1. Validates that the specified track is a metadata track (`type === 'meta'`)
2. Iterates through all `moof` boxes in the fragment
3. For each `moof`, finds `traf` boxes and filters by track ID
4. Parses `tfdt` box to get base decode time
5. Parses `tfhd` box to get default sample duration
6. Parses `trun` boxes to get sample timing, sizes, and offsets
7. Extracts raw sample data from `mdat` using calculated offsets
8. Creates `MetadataSample` objects with:
   - Correct PTS/DTS from base time + sample durations
   - Raw metadata as `Uint8Array`
   - Type set to `MetadataSchema.misbklv` (currently hardcoded for MISB KLV)

**Import changes:**

- Added imports for `MetadataSample` type and `MetadataSchema` enum from `../types/demuxer`

### 3. MP4 Demuxer (`src/demux/mp4demuxer.ts`)

**Added private fields (Lines: 39-40):**

```typescript
private initSegment?: Uint8Array;
private metaTrackId?: number;
```

**Modified `resetInitSegment()` method (Lines: 99-102):**

- Stores the init segment for later parsing
- Calls `findMetadataTrack()` to detect metadata tracks if enabled

**Added `findMetadataTrack()` method (Lines: 105-119):**

- Returns configured track ID if `dataTrackMetadataTrackId` is specified
- Otherwise auto-detects first track with `type === 'meta'`
- Returns `undefined` if no metadata track found

**Modified `demux()` method (Lines: 153-168):**

- After extracting EMSG metadata, checks if data track metadata is enabled
- If enabled and metadata track found, calls `parseDataTrackSamples()`
- Merges extracted samples with existing `id3Track.samples`

**Modified `flush()` method (Lines: 189-205):**

- Same logic as `demux()` to handle data track samples during flush

**Import changes:**

- Added `parseDataTrackSamples` to imports from `../utils/mp4-tools`

### 4. ID3 Track Controller (`src/controller/id3-track-controller.ts`)

**Lines: 206-209**

Modified `onFragParsingMetadata()` method to skip data track metadata samples:

**Data track metadata filtering:**

- Added check to skip samples with `type === MetadataSchema.misbklv`
- Data track metadata samples are only delivered via the `FRAG_PARSING_METADATA` event
- They are NOT added to the ID3 text track as VTTCues

**Existing behavior preserved:**

- ID3 and EMSG metadata continue to be processed as VTTCues
- No other changes to ID3TrackController functionality

## Usage Example

### Basic Usage (MISB KLV Example)

```javascript
const hls = new Hls({
  // Enable extraction from MP4 data tracks
  enableDataTrackMetadata: true,

  // Optional: Specify track ID (auto-detects first 'meta' track if omitted)
  // dataTrackMetadataTrackId: 3,
});

hls.attachMedia(video);
hls.loadSource('https://example.com/stream-with-data-track.m3u8');

// Listen for metadata via FRAG_PARSING_METADATA event
hls.on(Hls.Events.FRAG_PARSING_METADATA, (event, data) => {
  data.samples.forEach((sample) => {
    // Check if this is a MISB KLV sample (example use case)
    if (sample.type === 'urn:misb:KLV:bin:1910.1') {
      console.log('KLV sample:', {
        pts: sample.pts,
        dts: sample.dts,
        duration: sample.duration,
        data: sample.data, // Uint8Array of raw metadata
      });

      // Parse metadata according to your schema
      // Example: Extract MISB UAS Local Set (ST 0601)
      parseKLV(sample.data);
    }
  });
});
```

### Generic Data Track Metadata

```javascript
const hls = new Hls({
  enableDataTrackMetadata: true,
});

hls.on(Hls.Events.FRAG_PARSING_METADATA, (event, data) => {
  data.samples.forEach((sample) => {
    // All data track samples currently use type 'urn:misb:KLV:bin:1910.1'
    // In the future, this could be extended to support other types
    console.log('Data track metadata:', {
      pts: sample.pts,
      duration: sample.duration,
      rawData: sample.data,
    });

    // Parse according to your specific metadata format
    parseMetadata(sample.data);
  });
});
```

**Important Notes:**

- Data track metadata is delivered **only** via the `FRAG_PARSING_METADATA` event
- It is **not** added to the ID3 metadata text track as VTTCues
- You must listen to the `FRAG_PARSING_METADATA` event to receive samples
- Currently all samples use `MetadataSchema.misbklv` type - this can be extended in the future

## Technical Details

### MP4 Box Structure

The implementation parses the following MP4 box hierarchy:

```
moof (Movie Fragment)
├── traf (Track Fragment)
│   ├── tfhd (Track Fragment Header) - contains track ID, default duration
│   ├── tfdt (Track Fragment Decode Time) - contains base time
│   └── trun (Track Run) - contains sample count, sizes, durations, offsets
└── mdat (Media Data) - contains actual metadata sample data
```

### Timing Calculation

Sample timestamps are calculated as:

```
PTS = (baseTime / timescale) + timeOffset + (cumulativeDuration / timescale)
```

Where:

- `baseTime` comes from `tfdt` box
- `timescale` comes from init segment track info
- `timeOffset` is the fragment start time
- `cumulativeDuration` is the sum of previous sample durations from `trun`

### Data Extraction

Sample data is extracted from `mdat` using:

```
offset = moofOffset + dataOffset (from trun) + cumulativeSampleSizes
size = sampleSize (from trun for each sample)
```

## Testing

All tests passed:

- ✅ TypeScript compilation
- ✅ ESLint checks
- ✅ Build (all variants: full, light, worker, demo)
- ✅ API extraction

## Backwards Compatibility

The implementation is fully backwards compatible:

- Existing EMSG metadata support remains unchanged
- New features are disabled by default
- Both EMSG and data track metadata can work simultaneously
- No breaking changes to public API

## Performance Considerations

- Init segment is parsed once per segment in `resetInitSegment()`
- Init segment is re-parsed in `demux()` and `flush()` when data track metadata is enabled
  - Consider caching the parsed `initData` if performance is critical
- Data track parsing requires iterating through `moof`/`traf`/`trun` boxes
- More expensive than EMSG parsing due to sample-level iteration

## Future Enhancements

Potential improvements:

1. Cache parsed `initData` to avoid re-parsing in `demux()` and `flush()`
2. Add fourCC validation for specific metadata sample entry types
3. Support multiple metadata tracks
4. Make `MetadataSchema` type configurable instead of hardcoded to `misbklv`
5. Add configuration for different metadata standards (MISB, custom formats, etc.)
6. Integrate metadata parser libraries for automatic decoding

## Use Cases

This implementation supports any timed metadata stored in MP4 data tracks:

### MISB KLV Metadata

- **Use case:** Motion imagery metadata (drones, surveillance, etc.)
- **Standards:** MISB ST 0601, ST 0903, ST 1910.1
- **Sample type:** Currently `MetadataSchema.misbklv`

### Custom Metadata

- **Use case:** Application-specific timed data
- **Format:** Any binary format in MP4 data tracks
- **Sample type:** Currently all use `misbklv`, but can be extended

### GPS/Telemetry Data

- **Use case:** Location and sensor data synchronized with video
- **Format:** Custom binary or standard formats
- **Sample type:** Can be extended to support different schemas

## Standards Compliance

The implementation follows:

- **ISO/IEC 14496-12** (ISO Base Media File Format)
- **ISO/IEC 14496-30** (Timed text and other visual overlays)
- **MISB ST 1910.1** (Metadata KLV in Motion Imagery Files) - example use case

## Author Notes

This implementation was designed to provide general-purpose extraction of timed metadata from MP4 data tracks. While the current implementation hardcodes the metadata type to `MetadataSchema.misbklv` (suitable for MISB KLV metadata), the architecture is extensible and can support other metadata formats by:

1. Adding new entries to `MetadataSchema` enum
2. Detecting metadata type from fourCC or other indicators in the `stsd` box
3. Parsing metadata according to the detected schema

The raw binary data is exposed to applications via the `FRAG_PARSING_METADATA` event, allowing full flexibility in how the metadata is decoded and utilized.
