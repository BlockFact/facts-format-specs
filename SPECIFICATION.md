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

The `.facts` (Fact Stream) format is a binary container for live stream archives with cryptographic provenance. It uses an **interleaved segment/provenance architecture** — each segment of audio/video essence is immediately followed by its provenance record. This design ensures:

- Provenance is available during live capture (not just after stream ends)
- The file survives interruption (crash, power loss) without data loss
- Linear reading enables linear verification (no seeking required)
- Each segment is self-contained with its blockchain registration

## 2. Format Structure

```
┌──────────────────────────────────────────────────────────────────────┐
│ HEADER                                                               │
│ [4 bytes]  Magic: 0xFA 0x53 0x41 0x00                                │
│ [4 bytes]  Metadata length (big-endian unsigned 32-bit integer)       │
│ [N bytes]  JSON metadata (UTF-8 encoded)                             │
├──────────────────────────────────────────────────────────────────────┤
│ SEGMENT 0                                                            │
│ [4 bytes]  Segment marker: "SEG\0" (0x53 0x45 0x47 0x00)            │
│ [4 bytes]  Segment data length (big-endian u32)                      │
│ [S bytes]  Audio + video essence for this segment                    │
├──────────────────────────────────────────────────────────────────────┤
│ PROVENANCE 0                                                         │
│ [4 bytes]  Provenance marker: "PRV\0" (0x50 0x52 0x56 0x00)         │
│ [4 bytes]  Provenance data length (big-endian u32)                   │
│ [P bytes]  JSON provenance record for segment 0                      │
├──────────────────────────────────────────────────────────────────────┤
│ SEGMENT 1                                                            │
│ [4 bytes]  Segment marker: "SEG\0"                                   │
│ [4 bytes]  Segment data length (big-endian u32)                      │
│ [S bytes]  Audio + video essence                                     │
├──────────────────────────────────────────────────────────────────────┤
│ PROVENANCE 1                                                         │
│ [4 bytes]  Provenance marker: "PRV\0"                                │
│ [4 bytes]  Provenance data length (big-endian u32)                   │
│ [P bytes]  JSON provenance record for segment 1                      │
├──────────────────────────────────────────────────────────────────────┤
│ ... repeats for each segment ...                                     │
├──────────────────────────────────────────────────────────────────────┤
│ END-OF-STREAM (written when stream completes gracefully)             │
│ [4 bytes]  End marker: "END\0" (0x45 0x4E 0x44 0x00)                │
│ [4 bytes]  Footer length (big-endian u32)                            │
│ [F bytes]  JSON footer (summary, final stats, optional C2PA block)   │
└──────────────────────────────────────────────────────────────────────┘
```

## 3. Magic Bytes

| Byte | Value | Meaning |
|------|-------|---------|
| 0 | `0xFA` | BlockFact format family identifier |
| 1 | `0x53` | ASCII 'S' — Stream format |
| 2 | `0x41` | Format version (ASCII 'A' = version 1 of the block structure) |
| 3 | `0x00` | Reserved (must be zero) |

## 4. JSON Metadata (Header)

The metadata section is a UTF-8 encoded JSON object written at stream start:

```json
{
    "format_version": 2,
    "media_type": "stream",
    "stream_id": "550e8400-e29b-41d4-a716-446655440000",
    "started_at": "2026-06-30T12:00:00Z",
    "segment_duration_ms": 10000,
    "video_codec": "uyvy",
    "video_resolution": "1920x1080",
    "video_frame_rate": "29.97",
    "audio_sample_rate": 48000,
    "audio_channels": 2,
    "audio_bit_depth": 32,
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
| started_at | string | ISO 8601 timestamp of stream start |
| segment_duration_ms | integer | Target duration of each segment in milliseconds |
| wallet | string | Wallet address of the stream source |
| chain | string | Blockchain network identifier |

### Optional Fields

| Field | Type | Description |
|-------|------|-------------|
| video_codec | string | Video codec identifier |
| video_resolution | string | Resolution as "WIDTHxHEIGHT" |
| video_frame_rate | string | Frame rate as decimal or fraction |
| audio_sample_rate | integer | Audio sample rate in Hz |
| audio_channels | integer | Number of audio channels |
| audio_bit_depth | integer | Audio bit depth |
| contract | string | Smart contract address for registrations |
| transport_protocol | string | Original transport protocol (e.g. "ndi", "st2110", "srt") |
| source_name | string | Human-readable source identifier |
| obid_enabled | boolean | Whether OBID watermark is present in audio |
| stream_hash_version | integer | Hash algorithm version (for future compatibility) |

## 5. Segment Block

Each segment contains a fixed-duration portion of the stream's audio and video essence.

```
[4 bytes]  Marker: "SEG\0" (0x53 0x45 0x47 0x00)
[4 bytes]  Data length in bytes (big-endian u32)
[S bytes]  Raw audio + video essence for this time segment
```

The essence encoding format matches the stream's original format as specified in the header metadata. Segments are written sequentially as they are captured.

## 6. Provenance Block

Each provenance block immediately follows its corresponding segment and contains the blockchain registration record for that segment.

```
[4 bytes]  Marker: "PRV\0" (0x50 0x52 0x56 0x00)
[4 bytes]  Data length in bytes (big-endian u32)
[P bytes]  UTF-8 JSON provenance record
```

### Provenance Record Schema

```json
{
    "segment_index": 0,
    "content_hash": "0x1a2b3c4d5e6f...",
    "tx_hash": "0xabc123def456...",
    "parent_tx_hash": null,
    "wallet": "0x035e29...",
    "timestamp": "2026-06-30T12:00:00Z",
    "zkp_verified": true,
    "obid_reference": "0x1a2b3c4d5e6f7a8b9c0d1e2f"
}
```

| Field | Type | Description |
|-------|------|-------------|
| segment_index | integer | Sequential segment number (0-based) |
| content_hash | string | Poseidon hash of the segment's essence |
| tx_hash | string | Blockchain transaction hash for this registration |
| parent_tx_hash | string or null | Previous segment's tx_hash (null for genesis segment) |
| wallet | string | Signing wallet address |
| timestamp | string | ISO 8601 registration timestamp |
| zkp_verified | boolean | Whether ZKP was verified on-chain |
| obid_reference | string | Truncated hash embedded in OBID watermark (if enabled) |

### Provenance Timing

Because blockchain registration is asynchronous, the provenance block for segment N may be written:
- Immediately after segment N (if registration confirmed quickly)
- After segment N+1 has already been written (in which case the provenance block is inserted between segments)

Writers MUST ensure each provenance block follows its corresponding segment, even if out of order with subsequent segments. A valid sequence is:

```
SEG 0 → PRV 0 → SEG 1 → PRV 1 → SEG 2 → PRV 2
```

If registration for segment 1 takes longer than one segment duration:

```
SEG 0 → PRV 0 → SEG 1 → SEG 2 → PRV 1 → PRV 2
```

This is NOT valid. Writers MUST buffer segment 2 until provenance for segment 1 is written. Alternatively, writers MAY write a **pending provenance block** with `tx_hash: "pending"` and update it in-place when the transaction confirms.

## 7. End-of-Stream Footer

When a stream completes gracefully, an end marker and footer are appended:

```
[4 bytes]  Marker: "END\0" (0x45 0x4E 0x44 0x00)
[4 bytes]  Footer length in bytes (big-endian u32)
[F bytes]  UTF-8 JSON footer
```

### Footer Schema

```json
{
    "total_segments": 360,
    "duration_ms": 3600000,
    "ended_at": "2026-06-30T13:00:00Z",
    "genesis_tx_hash": "0xabc123...",
    "final_tx_hash": "0x789abc...",
    "chain_intact": true,
    "segments_verified": 360,
    "c2pa_manifest": "<base64-encoded JUMBF>"
}
```

The footer is optional — its absence indicates the stream was interrupted (crash, power loss, network failure). Readers MUST handle files without a footer gracefully by reading segment/provenance pairs until EOF.

## 8. Handling Interrupted Streams

If a stream is interrupted:
1. The file ends mid-segment or after the last complete provenance block
2. No END marker is present
3. All complete segment + provenance pairs written before interruption are valid
4. The last incomplete segment (if any) MAY be discarded by readers
5. The provenance chain up to the last complete pair remains verifiable

This is a key advantage of the interleaved design — nothing written is lost on interruption.

## 9. Tamper Detection

The interleaved provenance records enable verification:

1. Read each provenance block's `parent_tx_hash`
2. Verify it matches the previous provenance block's `tx_hash`
3. Any break in the chain indicates tampering, removal, or reordering
4. Optionally verify each `tx_hash` against the blockchain

Verification can proceed linearly — no seeking or index required.

## 10. OBID Watermark

When `obid_enabled: true` in the header, the audio essence in each segment contains an OBID watermark (per SMPTE ST 2112-10) in the 4-8kHz frequency band. The watermark carries a truncated blockchain reference (first 12 bytes of the previous segment's `tx_hash`) that any ST 2112-10 compliant decoder can extract.

This coexists with the proprietary BlockFact watermark in the 1-4kHz band, which carries the full Poseidon hash for offline self-contained verification.

## 11. Related Formats

| Format | MIME Type | Magic Bytes | Structure |
|--------|-----------|-------------|-----------|
| .facti | `image/vnd.blockfact.facti` | `FA 49 41 00` | Header + essence + appended extension blocks |
| .facta | `audio/vnd.blockfact.facta` | `FA 41 41 00` | Header + essence + appended extension blocks |
| .factv | `video/vnd.blockfact.factv` | `FA 56 41 00` | Header + essence + appended extension blocks |
| .facts | `application/vnd.blockfact.facts` | `FA 53 41 00` | Header + interleaved segments/provenance + optional footer |

The `.facts` format uses a different internal structure (interleaved) than the other formats (appended) because streams require write-during-capture and interruption resilience.

## 12. Security Considerations

- The `.facts` format does not execute code
- Provenance claims should be independently verified against the referenced blockchain
- Wallet addresses are pseudonymous blockchain identifiers
- The OBID audio watermark (if present) is imperceptible under normal listening conditions
- The interleaved design ensures partial files remain verifiable up to the point of interruption
- Writers SHOULD NOT store private keys or signing secrets in provenance records

## 13. IANA Registration

MIME type: `application/vnd.blockfact.facts`
Registration status: Pending
Contact: egbert@blockfact.io
Organization: BlockFact Technologies Inc

## 14. License

This specification is published under the MIT License.

Copyright (c) 2026 BlockFact Technologies Inc.
