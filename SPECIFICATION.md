# .facts Format Specification

**MIME Type:** `application/vnd.blockfact.facts`
**File Extension:** `.facts`
**Magic Bytes:** `0xFA 0x53 0x41 0x00`
**Status:** Draft
**Version:** 2.0
**Author:** Egbert von Frankenberg, BlockFact Technologies Inc
**Date:** 2026-06-30

---

## 1. Overview

The `.facts` (Fact Stream) format is a binary container for live stream archives with cryptographic provenance. It stores muxed audio and video essence alongside a daisy-chain provenance registry — an ordered sequence of blockchain-registered segment records that provide tamper-evident proof of what was streamed, when, and by whom.

## 2. Format Structure

```
┌──────────────────────────────────────────────────────────────────────┐
│ [4 bytes]  Magic: 0xFA 0x53 0x41 0x00                                │
│ [4 bytes]  Metadata length (big-endian unsigned 32-bit integer)       │
│ [N bytes]  JSON metadata (UTF-8 encoded)                             │
│ [M bytes]  Stream data (muxed audio + video segments)                │
│ ─── Extension Blocks (appended after stream data) ───                │
│ [2 bytes]  Extension block count (big-endian unsigned 16-bit integer)│
│ For each block:                                                      │
│   [4 bytes]  Block type (ASCII, e.g. "C2PA", "CHNR", "OBID", "WVRF")│
│   [4 bytes]  Block data length (big-endian unsigned 32-bit integer)   │
│   [L bytes]  Block data                                              │
└──────────────────────────────────────────────────────────────────────┘
```

## 3. Magic Bytes

| Byte | Value | Meaning |
|------|-------|---------|
| 0 | `0xFA` | BlockFact format family identifier |
| 1 | `0x53` | ASCII 'S' — Stream format |
| 2 | `0x41` | Format version (ASCII 'A' = version 1 of the block structure) |
| 3 | `0x00` | Reserved (must be zero) |

## 4. JSON Metadata

The metadata section is a UTF-8 encoded JSON object. Required fields:

```json
{
    "format_version": 2,
    "media_type": "stream",
    "stream_id": "550e8400-e29b-41d4-a716-446655440000",
    "total_segments": 360,
    "duration_ms": 3600000,
    "started_at": "2026-06-30T12:00:00Z",
    "ended_at": "2026-06-30T13:00:00Z",
    "segment_duration_ms": 10000,
    "video_codec": "uyvy",
    "video_resolution": "1920x1080",
    "video_frame_rate": "29.97",
    "audio_sample_rate": 48000,
    "audio_channels": 2,
    "audio_bit_depth": 32,
    "genesis_tx_hash": "0xabc123...",
    "final_tx_hash": "0x789abc...",
    "wallet": "0x035e29...",
    "chain": "starknet-mainnet",
    "contract": "0x028dc039...",
    "transport_protocol": "ndi",
    "source_name": "BlockFact Provenance Stream",
    "obid_enabled": true,
    "stream_hash_version": 1
}
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| format_version | integer | Must be 2 |
| media_type | string | Must be "stream" |
| stream_id | string | UUID identifying this stream session |
| total_segments | integer | Number of segments in the stream |
| duration_ms | integer | Total stream duration in milliseconds |
| started_at | string | ISO 8601 timestamp of stream start |
| segment_duration_ms | integer | Duration of each segment in milliseconds |
| genesis_tx_hash | string | Blockchain transaction hash of the first segment |
| wallet | string | Wallet address of the stream source |
| chain | string | Blockchain network identifier |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| ended_at | string | ISO 8601 timestamp of stream end (absent for live/ongoing) |
| video_codec | string | Video codec identifier |
| video_resolution | string | Resolution as "WIDTHxHEIGHT" |
| video_frame_rate | string | Frame rate as decimal or fraction |
| audio_sample_rate | integer | Audio sample rate in Hz |
| audio_channels | integer | Number of audio channels |
| audio_bit_depth | integer | Audio bit depth |
| final_tx_hash | string | Transaction hash of the last segment |
| contract | string | Smart contract address for registrations |
| transport_protocol | string | Original transport protocol (e.g. "ndi", "st2110", "srt") |
| source_name | string | Human-readable source identifier |
| obid_enabled | boolean | Whether OBID watermark is present in audio |
| stream_hash_version | integer | Hash algorithm version (for future compatibility) |

## 5. Stream Data

The stream data section contains muxed audio and video essence. The encoding format matches the original stream's format as indicated by the `video_codec` and `audio_sample_rate`/`audio_bit_depth`/`audio_channels` metadata fields.

The stream data is stored as sequential segments, each of `segment_duration_ms` duration.

## 6. Extension Blocks

Extension blocks are appended after the stream data. They follow the same structure as the `.facti` and `.facta` v2 formats.

### 6.1 Block Type: "C2PA"

Contains a C2PA JUMBF manifest store (signed). Provides standards-compliant provenance metadata verifiable by any C2PA-aware tool.

### 6.2 Block Type: "CHNR" (Chain Registry)

Contains the full daisy-chain provenance registry as a JSON array:

```json
[
    {
        "segment_index": 0,
        "content_hash": "0x1a2b3c...",
        "tx_hash": "0xabc123...",
        "parent_tx_hash": null,
        "wallet": "0x035e29...",
        "timestamp": "2026-06-30T12:00:00Z",
        "zkp_verified": true
    },
    {
        "segment_index": 1,
        "content_hash": "0x4d5e6f...",
        "tx_hash": "0xdef456...",
        "parent_tx_hash": "0xabc123...",
        "wallet": "0x035e29...",
        "timestamp": "2026-06-30T12:00:10Z",
        "zkp_verified": true
    }
]
```

Each entry links to its predecessor via `parent_tx_hash`, forming an unbreakable hash chain. The first segment has `parent_tx_hash: null`.

### 6.3 Block Type: "OBID"

Contains OBID (ST 2112-10) watermark metadata:

```json
{
    "obid_version": 1,
    "media_id_header": 2,
    "payload_type": "blockchain_provenance_reference",
    "frequency_band_hz": [4078, 8016],
    "embedding_standard": "SMPTE ST 2112-10:2020"
}
```

### 6.4 Block Type: "WVRF" (Wallet Verification)

Contains wallet verification challenge/response log:

```json
[
    {
        "timestamp": "2026-06-30T12:00:00Z",
        "challenger": "control-room-1",
        "nonce": "a7f9d32e...",
        "signature": "0x1b4a7c9d...",
        "verified": true
    }
]
```

## 7. Backward Compatibility

### Version 1 readers encountering Version 2 files

Version 1 readers that do not understand extension blocks will:
1. Read magic bytes (accepted — same family identifier)
2. Read metadata length and JSON (accepted — unknown fields ignored)
3. Treat remaining bytes as stream data

Stream data codecs (e.g., UYVY video, float PCM audio) are self-delineating or length-specified in metadata. Extension blocks appended after stream data are ignored.

### Version 2 readers encountering Version 1 files

Version 2 readers detect version 1 by absence of `format_version: 2` in JSON metadata. In this case, treat everything after metadata as stream data (no extension blocks).

## 8. Tamper Detection

The CHNR extension block enables offline verification:

1. For each entry, verify `parent_tx_hash` matches the previous entry's `tx_hash`
2. Any gap, reordering, or modification breaks the chain
3. Optionally verify each `tx_hash` against the blockchain for on-chain confirmation

## 9. Related Formats

| Format | MIME Type | Magic Bytes | Purpose |
|--------|-----------|-------------|---------|
| .facti | `image/vnd.blockfact.facti` | `FA 49 41 00` | Photographs with provenance |
| .facta | `audio/vnd.blockfact.facta` | `FA 41 41 00` | Audio recordings with provenance |
| .factv | `video/vnd.blockfact.factv` | `FA 56 41 00` | Video files with provenance |
| .facts | `application/vnd.blockfact.facts` | `FA 53 41 00` | Live stream archives with daisy-chain provenance |

All four formats share the same extension block architecture.

## 10. Security Considerations

- The `.facts` format does not execute code
- Provenance claims should be independently verified against the referenced blockchain
- Wallet addresses are pseudonymous blockchain identifiers
- The OBID audio watermark (if present) is imperceptible under normal listening conditions
- The CHNR block enables offline tamper detection without network access

## 11. IANA Registration

MIME type: `application/vnd.blockfact.facts`
Registration status: Pending
Contact: egbert@blockfact.io
Organization: BlockFact Technologies Inc

## 12. License

This specification is published under the MIT License.

Copyright (c) 2026 BlockFact Technologies Inc.
