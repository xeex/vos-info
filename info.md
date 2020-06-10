This guide will only concern itself with VOS022/VOS006 file format. Since all VOS files are binary, a hex editor such as [HxD](https://mh-nexus.de/en/hxd/) is recommended. In spirit of [the original guide](https://github.com/ReVanTis/VosDroid/wiki/Vos-File-Analysis), important bytes are denoted in lower-case letters `abc...`. Underscores `_` represent bytes that are unimportant (at least in the code block they are used in.)

# VOS vs. MIDI
All note files in VOS are treated in terms of "timing". As can be verified in VOS Creator, every measure consists of 3072 MTIME ticks, regardless of BPM. The position and duration of all VOS notes are written with this measurement framework, starting with 0 tick at the beginning.

This means the VOS file must obtain its BPM information from the MIDI segment (see below) and project the tickdiv (and possibly tempo) metadata onto the 3072-ticks-per-beat timing that VOS notes use.

A quick way to verify this theory would be to take two VOS charts with different MIDI BPMs but are otherwise identical. If the two binary files differ only in the MIDI segment, then this theory is correct. 

UPDATE: Tested with `test_faster.vow` and `shamisen.vow`, which have BPMs of 187 and 167 respectively. The only (non-trivial) difference between the two files was the BPM meta event `FF 51 03 aa aa aa`. Thus, **we can expect the theory above to be true.**

# Header

The header of a VOS022 file looks like this:
```
Offset(d)  00 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15
 00000000  02 00 00 00 0C 00 00 00 56 6F 73 63 74 65 6D 70   ........Vosctemp
 00000016  2E 74 72 6B aa aa aa aa 56 4F 53 30 32 32 __ __   .trk....VOS022..
```
`aa aa aa aa` supposedly represents the address for MIDI information, but I have yet to come across its significance, so vos.go currently ignores it.

Following this header, the metadata of the VOS file is stored in a contiguous structure. Each metadata is represented as two bytes for the little-endian $LENGTH of the data, followed by $LENGTH bytes of the data. For example, in `miyu.vow`, we see
```
Offset(d)  00 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15
 00000016  2E 74 72 6B 10 00 02 00 56 4F 53 30 32 32 06 00   .trk....VOS022..
 00000032  C8 ED C7 F7 B1 CD 07 00 55 6E 6B 6E 6F 77 6E 00   ÈíÇ÷±Í..Unknown.
```
The header ended at the 29th byte (32), so the next two bytes `{06 00}` represent the length of the title. In a little-endian architecture, `{06 00}` = 6, so the next 6 bytes should represent the title: `{C8 ED C7 F7 B1 CD}`[1]. The next two bytes `{07 00}` tell us there are 7 bytes in the next metadata: `{55 6E 6B 6E 6F 77 6E}`, which spells "Unknown" in ASCII.

The metadata are structured in the following sequence:
*  Title
*  Composer
*  Sequencer
*  VOS Charter
*  Genre

After the "Genre" metadata, the next two bytes represent the song type and volume respectively. The song type takes a value between 0 and 10 inclusively, and represents the following types.

| # | Type |
| - | ---- |
| 0 | POP |
| 1 | Korean POP |
| 2 | Japan POP |
| 3 | Rock |
| 4 | Metal |
| 5 | Jazz |
| 6 | Classic |
| 7 | New Age |
| 8 | TV & OST |
| 9 | Game & Anime |
| 10 | Other |

The volume takes a value between 0 and 100 (64 in hexadecimal).

The next four bytes are ignored.

The next four represent the "speed", or "노트속도" in VOS Creator. `00 aa 00 00`, where `aa` + 노트속도 value = 11. (or 0x0B)

The next byte is ignored.

The next four bytes represent total "MTIME" in little endian. `00 B4 4D 00` = `0x4db4` = 19892.

The next four represent total "RTIME" in little endian. `00 16 23 00` = `0x2316` = 8982 milliseconds.

The next 1024 bytes (0x400 in hexadecimal) compose the "Effect part" of the VOS file. Again, no clue what it's for (yet), but skipping 1024 bytes after time should take you to a byte sequence that resembles `aa aa aa aa 01 00 00 00`, which takes us to note data.

# Channel Information
The `aa aa aa aa` is the number of instruments used in the MIDI file. Following this pattern, each instrument will be expressed in the following way:
```04 bb bb bb bb```
where `bb bb bb bb` indicates the instrument.

In `miyu.vow`, we see the following, after lines of zero padding:
```
Offset(d)  00 01 02 03 04 05 06 07 08 09 10 11 12 13 14 15
 00000450  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 08   ................
 00000460  00 00 00 01 00 00 00 04 51 00 00 00 04 23 00 00   ........Q....#..
 00000470  00 04 6B 00 00 00 04 50 00 00 00 04 30 00 00 00   ..k....P....0...
 00000480  04 30 00 00 00 04 80 00 00 00 04 51 00 00 00 __   .0....€....Q....
```
The eight bytes starting from 08 (450:15) show that we have 8 instruments in total. We then see eight repetitions of the pattern `04 bb bb bb bb`, each representing an instrument.

The sequence `00 cc 0A 00 4D 69 78 65 64 20 4D 6F 64 65 00 00 00 00` comes afterward, and denotes the division between the instrument data and note data for the instruments; `cc` is the difficulty level. `00` for level 1, `01` for level 2, all the way to `09` for level 10.

# Note data
Now we will see `aa aa aa aa` blocks of binary data, corresponding to the notes played by each instrument. Each block takes the following form:
```
dd dd dd dd 00
[aa aa aa aa bb cc dd ee 01 ff gg gg gg gg hh 00]
```
where the segment in square brackets is repeated `dd dd dd dd` times.
* `a` = MTIME position
* `b` = Pitch
* `c` = Track
* `d` = Volume
* `e` = Played note? (0 for autoplay)
* `f` = Long note? (0 no, 1 yes)
* `g` = Sound length
* `h` = Whether it is played (00 no, FF yes)
Not sure what `h` represents at the moment, but a 0 in `e` is always matched with FF and 1 is matched with 00.
It is important to note that the `00` byte at the end of the square bracket segment acts as a divider between each note; it is missing in the last note definition.

# Blank data
At the header of the VOS file, it will specify whether the file is of VOS022 or a VOS006 format. If you are working with a VOS022 file, there is a 4-byte blank region `00 00 00 00` between the note data and the playing information. VOS006 files do not have this blank region, and the binary file transitions directly from note data to playing information.

# Playing information
The first 4 bytes `ee ee ee ee` determine the number of notes with playing information attached to them. Each playing note information takes the following form: `aa bb bb bb bb cc`, where
* `a` = Track
* `b` = Tone (within the track)
* `c` = Key (ranges from 0-6, presumably from left to right)

Thus, in total, the playing information section should take up 6 times `e` number of bytes. As an example, `teaser.vos` has `e=0xF0A`, which means the following `6 * 0xF0A = 0x5A3C` bytes will consist of the playing information.

# MIDI information
After the playing information segment, the byte sequence `00 00 00 00 00 00 00 00 0C 00 00 00 56 4F 53 43 54 45 4D 50 2E 6D 69 64 ff ff ff ff` can be observed. `f` represents the length of the MIDI information (but in a different sense, it represents the number of bytes left until EOF) and this sequence is followed by a partially completed MIDI file. The MIDI file generated from this section will contain all of the correct metadata and system events (MIDI title, number of tracks, instrument for each track, etc.), but **it will not have any actual musical data.** That is where the note data and playing information above come from, but at the moment, they require more research.

---
[1] This *should* spell 흡혈귀 in Korean, but I cannot find what encoding these are in. - Both EUC-KR or Windows 949 (newer) encoding handle this
