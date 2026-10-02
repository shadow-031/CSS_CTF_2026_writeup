# Signal Fracture - CTF Writeup

**Flag:** `CSSCTF{fragment_t3ll_th3_st0ry}`

## Challenge Description

KBR-17 briefly came back online for fourteen seconds before the Nexus fabric collapsed again. A passive network tap captured the relay's final uplink, while technicians recovered the relay's storage exactly as it was after the emergency shutdown.

Three details were given up front:

- The scheduler catalog survived.
- The spool index had been purged.
- Uplink retries may have produced duplicate copies of fragments.

The goal was to reconstruct the final message from these two pieces of forensic evidence.

## 1. Initial Enumeration

The challenge archive contained two files:

```
KBR17_relay.img
KBR17_uplink.pcap
```

First step was just unpacking everything and identifying what I was dealing with.

```bash
unzip Signal_Fracture.zip -d signal_fracture
cd signal_fracture
file KBR17_relay.img KBR17_uplink.pcap
```

The first file turned out to be a disk image, the second a packet capture. So from the start it was clear this challenge had two separate evidence sources that would need to be combined, not just one artifact to dig through.

## 2. Searching the Disk Image

Rather than mounting the image right away, I started by pulling strings out of the raw bytes, since a lot of CTF forensic images leave useful plaintext lying around even before you bother with a proper filesystem mount.

```bash
strings -a KBR17_relay.img | grep -Ei 'scheduler|spool|fragment|sha|emergency|burst|session'
```

This turned up some very useful log lines:

```
object=emergency_burst_17
session=6E2C17A9
fragments=8
```

along with:

```
fragment catalog committed
retry requested seq=6
retry acknowledged seq=6
scheduler state saved; uplink incomplete
emergency shutdown initiated
```

So at this point the target was clear:

```
Session:   6E2C17A9
Object:    emergency_burst_17
Fragments: 8
```

and it was already apparent that fragment 6 had been retransmitted at some point, which would matter later.

## 3. Discovering the Fragment Format

Digging further through the strings output, I found what looked like the actual spec for how fragments were structured on disk:

```
Nexus Transfer Manager - Fragment Record v3

0x00    4     magic = NXFR
0x04    4     session_id
0x08    2     sequence
0x0A    2     total_fragments
0x0C    4     payload_length
0x10    4     payload_crc32
0x14    ...   payload
```

That gave a reliable way to scan both the disk image and the pcap for fragment records, since every one starts with the four byte magic value `NXFR` followed by a predictable fixed header.

## 4. Recovering the Scheduler Catalog

The scheduler catalog mattered a lot here because it held the expected SHA-256 hash for every fragment, which is exactly what I'd need later to tell real fragments apart from stale duplicates.

Searching the image for the SQLite header:

```
SQLite format 3
```

located an embedded database starting at offset `0x69b000`, spanning five 4096 byte pages (20480 bytes total). I carved it out directly:

```bash
dd if=KBR17_relay.img \
   of=catalog.db \
   bs=1 \
   skip=$((0x69b000)) \
   count=$((5*4096))
```

Then opened it with Python to see what was inside:

```python
import sqlite3

db = sqlite3.connect("catalog.db")
print(db.execute(
    "SELECT name, sql FROM sqlite_master WHERE type='table'"
).fetchall())
```

The relevant table was `fragments`. Querying it:

```python
for row in db.execute("SELECT * FROM fragments"):
    print(row)
```

gave a full manifest, basically sequence number, expected SHA-256, and some fragment state or source info for each of the eight fragments.

## 5. Important Detail: The Hash Covers the Payload Only

One thing worth calling out clearly: the catalog specified that the SHA-256 value was computed over the payload only, not the whole NXFR record (header included). So for every candidate fragment, the comparison had to be:

```python
import hashlib
digest = hashlib.sha256(payload).hexdigest()
```

Hashing the full record instead of just the payload would silently produce the wrong hash and make a perfectly valid fragment look fake. This tripped me up briefly before I noticed the catalog's wording.

## 6. Recovering Fragments From the Relay Image

The spool index itself had been deleted, but that does not mean the underlying data was gone. This is a very common forensic situation: a deleted index just means the lookup table is gone, not that the raw records were wiped or overwritten.

```bash
strings -a KBR17_relay.img | grep NXFR
```

I used the magic bytes to locate candidate fragment headers, then parsed each one manually against the format from step 3. The image contained fragments for:

```
seq 0
seq 1
seq 2
seq 2
seq 4
seq 5
```

Sequence 2 showed up twice. The challenge description had already warned that retries could leave duplicate copies behind, so this was expected rather than alarming.

## 7. Resolving the Duplicate Sequence 2

The two candidates for sequence 2 hashed differently:

```
Candidate A: fdb199cac2766f1e5887025ca777cfaee6df65b0c868a8e3f04785dff499885d
Candidate B: 5cd4fd1f9bdc1360dc272362ee421b4f62f2818c7d68332440042c3e46b8f732
```

The scheduler catalog expected:

```
5cd4fd1f9bdc1360dc272362ee421b4f62f2818c7d68332440042c3e46b8f732
```

So candidate A was rejected and candidate B was kept. The sequence number alone was not enough to settle which copy was correct, only the catalog's hash could do that.

## 8. Analyzing the Network Capture

The second evidence source was the pcap. I filtered for the NXFR magic and the target session ID:

```
session = 6E2C17A9
```

which turned up:

```
seq 3
seq 6
seq 6
seq 7
```

Sequence 6 again appeared twice, matching the earlier log entries about a retry being requested and acknowledged for that same sequence number.

## 9. Resolving the Sequence 6 Retry

The two sequence 6 candidates had different payload hashes:

```
First candidate:  a6b0308e9c159807f6ff458b6121dd49d9d0184f39c58ce78d59221794d4879f
Second candidate: fa9dbecb99768ea4d8470546b270845127278e797e09f71073df7e5658bdabae
```

The catalog expected:

```
fa9dbecb99768ea4d8470546b270845127278e797e09f71073df7e5658bdabae
```

So the first candidate was rejected and the second kept. If I had just grabbed whichever packet appeared first in the capture, the reconstruction would have failed silently later on.

## 10. Final Fragment Map

After checking every candidate against the catalog, the final map of where each fragment came from looked like this:

| Sequence | Source      | Result                     |
|---------:|-------------|-----------------------------|
| 0        | Relay image | Accepted                   |
| 1        | Relay image | Accepted                   |
| 2        | Relay image | Correct duplicate selected |
| 3        | PCAP        | Accepted                   |
| 4        | Relay image | Accepted                   |
| 5        | Relay image | Accepted                   |
| 6        | PCAP        | Correct retry selected     |
| 7        | PCAP        | Accepted                   |

Notice that the full message needed both sources. Neither the disk image nor the pcap held all eight fragments on its own.

## 11. Reassembling the Payload

Fragments were ordered by sequence number and only the payload bytes were concatenated, no headers:

```python
with open("reassembled.gz", "wb") as out:
    for seq in range(8):
        out.write(fragment[seq])
```

The resulting file, `reassembled.gz`, came out to 6725 bytes.

## 12. Identifying the Compression Format

The first three bytes of the reassembled file were:

```
1f 8b 08
```

which is the standard gzip magic. I confirmed it with:

```bash
gzip -t reassembled.gz
```

The test passed, which was a good sign that the fragments had been ordered and selected correctly. If the wrong sequence-2 or sequence-6 candidate had been used instead, this test would have failed.

## 13. Decompressing the Stream

```bash
gzip -dc reassembled.gz > reassembled.tar
file reassembled.tar
```

This identified the decompressed data as a POSIX tar archive, so the full structure of the data turned out to be: NXFR fragments wrapped in gzip, wrapped in tar.

## 14. Inspecting the TAR Archive

```bash
tar -tvf reassembled.tar
```

listed three files:

```
final_message.txt
burst_metadata.json
telemetry.bin
```

Extracted everything with:

```bash
mkdir final
tar -xf reassembled.tar -C final
```

## 15. Verifying the Metadata

```bash
cat final/burst_metadata.json
```

confirmed the following, which matched what was already pulled from the scheduler database earlier:

```
relay = KBR-17
object = emergency_burst_17
fragment_count = 8
```

This cross check was reassuring since it meant the reconstruction lined up with independent metadata rather than just passing the gzip and tar sanity checks by chance.

## 16. Recovering the Final Message

```bash
cat final/final_message.txt
```

revealed the recovered message along with the flag.

## 17. Final Answer

```
CSSCTF{fragment_t3ll_th3_st0ry}
```

## 18. Key Takeaways

**Deleted indexes do not mean deleted data.** The spool index was purged, but the raw fragment records were still sitting on disk and recoverable by scanning for the magic bytes directly.

**Sequence numbers alone are not a reliable ordering key.** Both sequence 2 and sequence 6 had duplicate copies in this challenge, and only the scheduler's SHA-256 manifest could tell which copy was legitimate.

**No single evidence source told the whole story.** The disk image and the pcap each held a different, overlapping subset of the eight fragments. The full message only came together once both were combined.

**Validate at every stage rather than trusting the first thing that looks right.** The full validation chain here was: NXFR header check, CRC32, SHA-256 against the scheduler catalog, correct sequence ordering, a valid gzip stream, a valid tar archive, and finally metadata that matched earlier findings. Any one of these checks failing would have been a signal that a wrong fragment had been picked somewhere upstream.

## Evidence Chain Summary

```
Signal_Fracture.zip
    |
    |-- KBR17_relay.img
    |       |-- logs
    |       |-- scheduler SQLite DB
    |       `-- raw NXFR fragments
    |
    `-- KBR17_uplink.pcap
            `-- NXFR fragments + retry

                    |
                    v

        Session: 6E2C17A9
        Object:  emergency_burst_17
        Fragments: 8

                    |
                    v

            SHA-256 verification

                    |
                    v

              0 1 2 3 4 5 6 7

                    |
                    v

              gzip stream -> tar archive -> final_message.txt

                    |
                    v

        CSSCTF{fragment_t3ll_th3_st0ry}
```
