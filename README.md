# Karaoke Music File (`.kmf`)

A simple binary container that bundles **audio** and **enhanced LRC lyrics** into a single file, designed for karaoke applications.

---

## Table of contents

**For everyone**
- [What is a KMF file?](#what-is-a-kmf-file)
- [What's inside?](#whats-inside)
- [Why use it?](#why-use-it)

**For developers**
- [Technical specification](#technical-specification)
- [License](#license)

---

# For everyone

## What is a KMF file?

Normally, karaoke needs two separate files: a music file and a lyrics file. You probably know the pain of searching for lyrics that are synced with your audio file.

A KMF file solves this by putting both into one package.

## What's inside?

- **The music.** The actual song, probably an MP3 or OGG audio file.
- **Timed lyrics.** The words of the song, with timing so app can highlight each word right when you should sing it.
- **Lyrics for many singers** (v1+). Duets and group songs can include a separate set of lyrics per person, so each singer sees only their own parts.
- **Singer names** (v2+). Instead of "Track 1" and "Track 2", the app can show for example "Female Soprano" or "Male Baritone".
- **Different lyrics formats** (v2+). Lyrics can come in plain timed text or fancier styles with colors and effects.

> **Good to know:** a file holds up to 255 lyrics sets, singer names can be up to 255 bytes long, and each set of lyrics can be up to about 4 GB. In practice, you won't hit these limits.

## Why use it?

| Benefit | What it means for you |
|---------|-----------------------|
| **Everything stays together** | No more lost or mismatched lyrics files |
| **Easy to share** | Send one file to a friend and it just works |
| **Great for duets and groups** | Each singer can have their own lyrics |
| **Flexible** | Works with different audio formats and lyrics styles |
| **Open and free** | Anyone can build apps and tools for it (MIT License) |


# For developers

## Technical specification

| | |
|---|---|
| **Extension** | `.kmf` |
| **Magic header** | `==kmusf==` (ASCII, 9 bytes) |
| **Byte order** | Little-endian |
| **Text encoding** | UTF-8 |
| **Current version** | 2 |

*A "magic header" is a fixed piece of text at the very start of a file. Apps check it to confirm they are really looking at a KMF file.*

### Conventions

| Notation | Meaning |
|----------|---------|
| `u8`, `u16`, `u32` | Unsigned integer of 8/16/32 bits, **little-endian** |
| `string` | UTF-8 encoded text, **without** a terminator |
| `u8[]` | Raw byte sequence |
| `?` | Variable size, determined by a preceding length field |

---

### Version overview

| Version | Lyrics | Audio | Status |
|:-------:|--------|-------|--------|
| 0 | 1 block | 1 track (rest of file) | Legacy |
| 1 | N blocks (one per singer) | 1 track (rest of file) | Legacy |
| 2 | N blocks (one per singer), each with a name and a format | 1 track (rest of file) | **Current** |

---

### Version 0

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

### Version 1

Same as version 0, but the single lyrics block is replaced by **N blocks, one per singer**.

#### Header

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| 9 | string | `magic` | Magic header `==kmusf==` |
| 2 | u16 | `version` | Format version (`1`) |
| 1 | u8 | `format_len` | Length of the audio format string |
| ? | string | `format` | Audio format, e.g. `mp3` |
| 1 | u8 | `lyrics_count` | Number of lyrics blocks (**N**) |

#### Lyrics blocks (repeated **N** times)

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| 4 | u32 | `lyrics_len` | Length of this block's enhanced LRC lyrics |
| ? | string | `lyrics` | This block's enhanced LRC lyrics |

#### Audio

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| ? | u8[] | `audio` | Audio data (until end of file) |

---

### Version 2

Same as version 1, but each lyrics block now carries additional metadata: a **name** (to display in the application instead of a generic label such as "Track 1") and a **format** describing how the lyrics are stored.

#### Header

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| 9 | string | `magic` | Magic header `==kmusf==` |
| 2 | u16 | `version` | Format version (`2`) |
| 1 | u8 | `format_len` | Length of the audio format string |
| ? | string | `format` | Audio format, e.g. `mp3` |
| 1 | u8 | `lyrics_count` | Number of lyrics blocks (**N**) |

#### Lyrics blocks (repeated **N** times)

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| 1 | u8 | `lyrics_name_len` | Length of the lyrics name. `0` means no name |
| ? | string | `lyrics_name` | Name of the channel/singer, shown in the application instead of "Track 1", "Track 2", etc. Absent if `lyrics_name_len == 0` |
| 1 | u8 | `lyrics_format_len` | Length of the lyrics format string. `0` means the format is `elrc` |
| ? | string | `lyrics_format` | Format of the lyrics, e.g. `elrc`, `ass`, `srt`. Absent if `lyrics_format_len == 0` |
| 4 | u32 | `lyrics_len` | Length of this block's lyrics |
| ? | string | `lyrics` | This block's lyrics, encoded as described by `lyrics_format` |

##### Field defaults

| Field | Condition | Behavior |
|-------|-----------|----------|
| `lyrics_name` | `lyrics_name_len == 0` | No name is set. Applications may fall back to a generic label such as "Track *n*" |
| `lyrics_format` | `lyrics_format_len == 0` | Assume `elrc` (enhanced LRC) |

#### Audio

| Size (bytes) | Type | Field | Description |
|:------------:|:----:|-------|-------------|
| ? | u8[] | `audio` | Audio data (until end of file) |

### Examples

#### Version 1

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

#### Version 2

Hexdump of the beginning of a **version 2** file with `mp3` audio and one lyrics block named `Ann`, with explicit format `srt`, containing `hello`:

```text
3D 3D 6B 6D 75 73 66 3D 3D   magic             "==kmusf=="
02 00                        version           2
03                           format_len        3
6D 70 33                     format            "mp3"
01                           lyrics_count      1
03                           lyrics_name_len   3
41 6E 6E                     lyrics_name       "Ann"
03                           lyrics_format_len 3
73 72 74                     lyrics_format     "srt"
05 00 00 00                  lyrics_len        5
68 65 6C 6C 6F               lyrics            "hello"
...                          audio             (rest of file)
```

The same block with no name and the default format (`elrc`) is encoded more compactly:

```text
00                           lyrics_name_len   0   (no name)
00                           lyrics_format_len 0   (assume "elrc")
05 00 00 00                  lyrics_len        5
68 65 6C 6C 6F               lyrics            "hello"
```

### Parsing guidelines

1. Read the 9-byte magic header and verify it equals `==kmusf==`. Reject the file otherwise.
2. Read the `u16` version and select the matching parser. Reject unsupported versions.
3. **Validate every declared length** against the number of remaining bytes before reading.
4. Read strings using the declared byte length and decode them as UTF-8.
5. *(v2)* For each lyrics block, if `lyrics_name_len == 0` treat the block as unnamed; if `lyrics_format_len == 0` treat the format as `elrc`.
6. *(v2)* Handle unknown `lyrics_format` values gracefully (e.g. skip the block or show it as unsupported) rather than rejecting the whole file. The `lyrics_len` field always allows skipping a block.
7. Treat everything after the last lyrics block (v1, v2) or the lyrics block (v0) as audio data.

---

## License

This project is licensed under the [MIT License](LICENSE).
