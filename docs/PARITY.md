# pyatv parity audit

Source audit on 2026-09-28. Compared this fork's inherited Rust implementation at [`7a56263`](https://github.com/SkrOYC/pyatv-rs/tree/7a56263) with Python [`pyatv` at `b277a4c`](https://github.com/postlund/pyatv/tree/b277a4c). This is a code comparison, not a claim that every feature works on every device.

The Rust workspace wires discovery, pairing, and connection into implementations of the main remote-control, metadata, streaming, power, apps, accounts, audio, keyboard, and touch interfaces. It contains Companion, MRP, AirPlay, RAOP, and DMAP protocol crates. The earlier [`step8-parity-gap-analysis.md`](research/step8-parity-gap-analysis.md) predates much of that implementation and should not be used as the current gap list.

## Confirmed gaps

| Area | Difference from Python pyatv | Evidence |
| --- | --- | --- |
| RAOP network audio | Rust rejects HTTPS audio URLs, downloads an HTTP source fully before decoding, and limits downloads to 256 MiB and decoded audio to 1 GiB. Python accepts HTTPS and decodes network sources incrementally. Live radio and long files therefore have different behavior. | [Rust URL handling](../crates/pyatv-proto-airplay/src/audio/fetch/url.rs), [Rust fetcher](../crates/pyatv-proto-airplay/src/audio/fetch.rs), [Python source](https://github.com/postlund/pyatv/blob/b277a4c/pyatv/protocols/raop/audio_source.py#L458) |
| AirPlay `play_url` | Python can serve a local file path to the TV and expose a playback start position. The Rust facade passes its input as a URL and starts at zero. A lower-level Rust `play_url_at` method exists but is not exposed through the common `Stream` interface. | [Python AirPlay](https://github.com/postlund/pyatv/blob/b277a4c/pyatv/protocols/airplay/__init__.py#L106), [Rust AirPlay](../crates/pyatv-proto-airplay/src/setup/interfaces.rs), [Rust `Stream` trait](../crates/pyatv-core/src/interface/playback.rs) |
| Discovery and settings | Python `scan` accepts caller-provided Zeroconf and storage and applies saved settings to discovered configurations. Rust `ScanOptions` has no equivalents. Python exposes settings on a connected `AppleTV`; Rust exposes storage separately. | [Python scan](https://github.com/postlund/pyatv/blob/b277a4c/pyatv/__init__.py#L33), [Rust scan options](../crates/pyatv-mdns/src/browse.rs), [Rust `AppleTV` trait](../crates/pyatv-core/src/interface.rs) |
| Pairing name | Python allows a caller-provided controller name during pairing. Rust's public pairing path uses the Companion default and does not send the optional AirPlay HAP Name field. | [Python pairing](https://github.com/postlund/pyatv/blob/b277a4c/pyatv/protocols/companion/pairing.py#L19), [Rust pairing](../crates/pyatv/src/pair.rs), [Rust AirPlay HAP](../crates/pyatv-proto-airplay/src/auth/hap.rs) |
| `atvremote` CLI | Rust lacks Python's interactive setup wizard, chained commands, and manual `--service-properties` option. Most individual controls, settings commands, touch gestures, and JSON output are present. | [Rust CLI](../cli/atvremote/src/cli/command.rs), [Rust manual setup](../cli/atvremote/src/commands.rs), [Python CLI](https://github.com/postlund/pyatv/blob/b277a4c/pyatv/scripts/atvremote.py#L242) |

## Hardware coverage and shared failures

The inherited [live validation report](research/live-parity-validation-2026-08-25.md) used one Apple TV 4K running tvOS 27. It did not validate direct MRP on older tvOS, legacy DMAP or AirPlay devices, HomePod transient pairing, or multiple output speakers. On that Apple TV, AirPlay video playback and RAOP audio streaming failed with both the Rust port and Python pyatv; those observations do not establish a Rust-specific regression.

For this fork, the next device-level gate is pairing and exercising the remote, app launching, text entry, and now-playing updates on our own Apple TV before treating the inherited validation as sufficient.
