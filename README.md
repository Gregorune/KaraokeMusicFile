# Karaoke Music File (`.kmf`)

A simple binary container that bundles **audio** and **enhanced LRC lyrics** into a single file, designed for karaoke applications.

| | |
|---|---|
| **Extension** | `.kmf` |
| **Magic header** | `==kmusf==` (ASCII, 9 bytes) |
| **Byte order** | Little-endian |
| **Text encoding** | UTF-8 |
| **Current version** | 1 |

---

## Table of contents

- [Conventions](#conventions)
- [Version overview](#version-overview)
- [Version 0](#version-0)
- [Version 1](#version-1)
- [Example](#example)
- [Parsing guidelines](#parsing-guidelines)
- [License](#license)

---

## Conventions

| Notation | Meaning |
|----------|---------|
| `u8`, `u16`, `u32` | Unsigned integer of 8/16/32 bits, **little-endian** |
| `string` | UTF-8 encoded text, **without** a terminator |
| `u8[]` | Raw byte sequence |
| `?` | Variable size, determined by a preceding length field |

---

## Version overview

| Version | Lyrics | Audio | Status |
|:-------:|--------|-------|--------|
| 0 | 1 block | 1 track (rest of file) | Legacy |
| 1 | N blocks (one per singer) | 1 track (rest of file) | **Current** |

---

## Version 0

One lyrics block followed by the audio data.

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| 9 | string | `magic` | Magic header `==kmusf==` |
| 2 | u16 | `version` | Format version (`0`) |
| 1 | u8 | `format_len` | Length of the audio format string |
| ? | string | `format` | Audio format, e.g. `mp3` |
| 4 | u32 | `lyrics_len` | Length of the enhanced LRC lyrics |
| ? | string | `lyrics` | Enhanced LRC lyrics |
| ? | u8[] | `audio` | Audio data (until end of file) |

---

## Version 1

Same as version 0, but the single lyrics block is replaced by **N blocks, one per singer**.

### Header

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| 9 | string | `magic` | Magic header `==kmusf==` |
| 2 | u16 | `version` | Format version (`1`) |
| 1 | u8 | `format_len` | Length of the audio format string |
| ? | string | `format` | Audio format, e.g. `mp3` |
| 1 | u8 | `lyrics_count` | Number of lyrics blocks (**N**) |

### Lyrics blocks (repeated **N** times)

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| 4 | u32 | `lyrics_len` | Length of this block's enhanced LRC lyrics |
| ? | string | `lyrics` | This block's enhanced LRC lyrics |

### Audio

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| ? | u8[] | `audio` | Audio data (until end of file) |

---

## Example

Hexdump of the beginning of a **version 1** file with `mp3` audio and one lyrics block containing `hello`:

```text
3D 3D 6B 6D 75 73 66 3D 3D   magic        "==kmusf=="
01 00                        version      1
03                           format_len   3
6D 70 33                     format       "mp3"
01                           lyrics_count 1
05 00 00 00                  lyrics_len   5
68 65 6C 6C 6F               lyrics       "hello"
...                          audio        (rest of file)
```

---

## Parsing guidelines

1. Read the 9-byte magic header and verify it equals `==kmusf==`. Reject the file otherwise.
2. Read the `u16` version and select the matching parser. Reject unsupported versions.
3. **Validate every declared length** against the number of remaining bytes before reading.
4. Read strings using the declared byte length and decode them as UTF-8.
5. Treat everything after the last lyrics block (v1) or the lyrics block (v0) as audio data.

---

## License

This project is licensed under the [MIT License](LICENSE).
