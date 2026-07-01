# .facts Format Specification

The `.facts` (Fact Stream) format is a binary container for live stream archives with built-in cryptographic provenance. It provides tamper-evident proof of what was streamed, when, and by whom — using blockchain-registered daisy-chain segment records.

## MIME Type

`application/vnd.blockfact.facts`

## File Extension

`.facts`

## Magic Bytes

`0xFA 0x53 0x41 0x00`

## Specification

See [SPECIFICATION.md](SPECIFICATION.md) for the full format definition.

## Related Formats

- [.facti](https://github.com/BlockFact/facti-format-spec) — Images with provenance (`image/vnd.blockfact.facti`)
- .facta — Audio with provenance (`audio/vnd.blockfact.facta`)
- .factv — Video with provenance (`video/vnd.blockfact.factv`)

## Key Features

- **Daisy-chain provenance**: Sequential segments linked via blockchain transaction hashes
- **Tamper detection**: Removing, reordering, or modifying any segment breaks the chain
- **Dual watermark**: Proprietary (1-4kHz) + OBID ST 2112-10 (4-8kHz) in audio
- **C2PA compatible**: Standard JUMBF manifest in extension block
- **Protocol agnostic**: Works with NDI, ST 2110, SRT, WebRTC, HLS/DASH
- **Wallet verification**: Bidirectional challenge-response for source authentication

## License

MIT License — Copyright (c) 2026 BlockFact Technologies Inc.
