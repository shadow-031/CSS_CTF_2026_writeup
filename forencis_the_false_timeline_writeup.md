# The False Timeline - CTF Writeup

**Category:** Forensics

## Challenge Overview

The challenge provides a single disk image, `nexus_relay.img`, tied to a relay server named `nx-17`. The premise is that an emergency data export occurred on this machine, and the server's own application log attributes that export to a user called KAI-7. The task is to determine whether that attribution actually holds up, and if not, to work out what really happened and recover the stolen key that was exported.

## 1. Identify the Filesystem

First step was figuring out what I was actually looking at.

```bash
file nexus_relay.img
fdisk -l nexus_relay.img
```

Both confirmed that the entire 128 MiB image is a single ext4 filesystem, no partition table wrapping it. Running `parted` as well just to be sure:

```
Partition Table: loop

Number  Start  End    Size   File system
1      0.00B  134MB  134MB  ext4
```

Since this is a loop partition table with one filesystem filling the whole image, there was no offset to calculate before mounting. I mounted it read only, which is standard practice so nothing in the evidence gets accidentally modified:

```bash
sudo mkdir /mnt/nexus
sudo mount -o loop,ro nexus_relay.img /mnt/nexus
```

## 2. Enumerate the Filesystem

With the image mounted, I did a full file listing to see what was actually on it:

```bash
sudo find /mnt/nexus -type f -printf '%p %s bytes\n' 2>/dev/null
```

A handful of files stood out immediately:

```
/etc/passwd
/home/kai/.bash_history
/opt/nexus/bin/nx-export
/opt/nexus/lib/exporter.py
/var/cache/nexus/.ekey-cache
/var/lib/nexus/metadata.db
/var/log/audit/audit.log
/var/log/auth.log
/var/log/nexus/relay.log
```

The most interesting one to start with was Kai's own shell history:

```
sudo systemctl status nexus-relay
journalctl -u nexus-relay
ls -la /var/log/nexus/
cat /etc/nexus/relay.conf
```

This told me Kai had been poking around the nexus relay service shortly before whatever happened, which made the relay logs and service behavior the obvious next place to look.

## 3. Examine the Authentication Log

```bash
cat /mnt/nexus/var/log/auth.log
```

The relevant entries were:

```
Oct  1 02:52:11 nx-17 sshd[1822]: Accepted publickey for kai from 10.5.21.99
Oct  1 03:05:41 nx-17 sshd[1822]: pam_unix(sshd:session): session closed for user kai
Oct  1 03:10:02 nx-17 su[1890]: Successful su for svc-relay by root
```

This is where things started to look suspicious. Kai's SSH session closed at 03:05:41. A separate session for the `svc-relay` account was opened about four and a half minutes later, at 03:10:02. But the relay application log claims the emergency export happened around 03:17 and attributes it to KAI-7.

That timing gap is the whole crux of the challenge. Kai was not even logged in anymore when the export supposedly happened under his name. That discrepancy meant the relay log's attribution could not be trusted at face value and needed to be checked against something more authoritative.

## 4. Examine the Audit Log

```bash
cat /mnt/nexus/var/log/audit/audit.log
```

A couple of entries mattered here. First, Kai's earlier status check showed up as expected:

```
comm="systemctl"
exe="/usr/bin/systemctl"
```

More importantly, the actual export process appeared as its own audit record:

```
comm="nx-export"
exe="/opt/nexus/bin/nx-export"
key="nexus_export"

argc=2
a0="/opt/nexus/bin/nx-export"
a1="--emergency"
```

and crucially, the UID attached to that process was:

```
uid=998
gid=998
euid=998
```

Checking `/etc/passwd` for UID 998:

```
svc-relay:x:998:998:Nexus Relay Service:/var/lib/nexus:/bin/false
```

So the process that actually ran `nx-export --emergency` was running as `svc-relay`, a service account, not as Kai. The audit log at the kernel level is a much stronger source of truth here than an application log, since it records who the process actually ran as rather than who an application chose to log.

Putting the timeline together so far:

```
02:52:11  KAI-7 logs in
02:55:01  KAI-7 checks nexus-relay
03:05:41  KAI-7 logs out
03:10:02  svc-relay session begins
03:17     emergency export occurs
```

## 5. Find the Authoritative Event Information

Next I went looking for anything that could confirm the export details independently of both the relay log and the audit log. The filesystem had:

```
/var/lib/nexus/metadata.db
```

```bash
file /mnt/nexus/var/lib/nexus/metadata.db
```

confirmed it was a SQLite database. Listing its tables:

```bash
sqlite3 /mnt/nexus/var/lib/nexus/metadata.db '.tables'
```

returned:

```
jobs      sessions
```

Dumping the whole database:

```bash
sqlite3 /mnt/nexus/var/lib/nexus/metadata.db '.dump'
```

produced the key records:

```sql
INSERT INTO sessions VALUES(
'aab0c8b2-f8b1-4f11-9a72-6d8123a1005a',
'svc-relay',
'127.0.0.1',
4157543482,
NULL
);

INSERT INTO jobs VALUES(
2,
'aab0c8b2-f8b1-4f11-9a72-6d8123a1005a',
'emergency_export',
'emergency-core',
'completed',
4157543568
);
```

This was the real evidence I needed. It ties the emergency export job directly to a `svc-relay` session, with a specific session UUID and a completion epoch:

```
Principal:     svc-relay
Session UUID:  aab0c8b2-f8b1-4f11-9a72-6d8123a1005a
Event epoch:   4157543568
```

Between this database record and the audit log's UID, there were now two independent sources agreeing that `svc-relay` performed the export, against one single source (the relay application log) claiming it was Kai. At that point the KAI-7 attribution was effectively ruled out.

## 6. Examine the Encrypted Key

The next file worth looking at was:

```
/var/cache/nexus/.ekey-cache
```

```bash
xxd /mnt/nexus/var/cache/nexus/.ekey-cache
```

gave:

```
00000000: 4e58 4b45 595f 5631 ...
```

Those first bytes decode to the ASCII string `NXKEY_V1`. So this was not a plaintext file at all, it was some kind of custom encrypted key package with its own header format.

## 7. Find the Encryption Algorithm

Conveniently, the image also included the exporter's source code:

```
/opt/nexus/lib/exporter.py
```

```bash
cat /mnt/nexus/opt/nexus/lib/exporter.py
```

The relevant function was:

```python
def derive_emergency_key(machine_id: str,
                          session_uuid: str,
                          event_epoch: int) -> bytes:

    material = f"{machine_id.strip()}|{session_uuid.strip()}|{int(event_epoch)}"

    return hashlib.sha256(
        material.encode('utf-8')
    ).digest()
```

and the actual encryption step:

```python
key = derive_emergency_key(
    machine_id,
    session_uuid,
    event_epoch
)

aesgcm = AESGCM(key)
nonce = os.urandom(12)

ciphertext = aesgcm.encrypt(
    nonce,
    plaintext,
    None
)

return b'NXKEY_V1' + nonce + ciphertext
```

This answered everything about the format. The key is a SHA-256 hash over three pipe-separated fields, machine ID, session UUID, and event epoch. The file itself is just that `NXKEY_V1` tag, a 12 byte nonce, and then the AES-GCM ciphertext with its authentication tag appended.

I already had the session UUID and event epoch from the database. The only missing piece was the machine ID.

## 8. Recover the Machine ID

```bash
cat /mnt/nexus/etc/machine-id
```

returned:

```
8f3b2a1c9e4d56781234abcd567890ef
```

So all three inputs required by `derive_emergency_key` were now known:

```
machine_id:   8f3b2a1c9e4d56781234abcd567890ef
session_uuid: aab0c8b2-f8b1-4f11-9a72-6d8123a1005a
event_epoch:  4157543568
```

## 9. Reconstruct the AES Key

Following the exporter code exactly, the material string to hash is:

```
8f3b2a1c9e4d56781234abcd567890ef|aab0c8b2-f8b1-4f11-9a72-6d8123a1005a|4157543568
```

Running SHA-256 over that string produces a 32 byte digest, which is used directly as the AES-256 key. It was important to match the formatting exactly, same stripping, same pipe separators, same integer conversion for the epoch, since any difference there produces a completely different key and the decryption would simply fail.

## 10. Decrypt the Cache

Recalling the file layout from the exporter source:

```
NXKEY_V1        (8 bytes, magic tag)
nonce           (12 bytes)
ciphertext      (remaining bytes, includes GCM auth tag)
```

I wrote a short script to reproduce the key and decrypt the cache file:

```python
import hashlib
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

machine_id = "8f3b2a1c9e4d56781234abcd567890ef"
session_uuid = "aab0c8b2-f8b1-4f11-9a72-6d8123a1005a"
event_epoch = 4157543568

material = f"{machine_id}|{session_uuid}|{event_epoch}"
key = hashlib.sha256(material.encode()).digest()

data = open(
    "/mnt/nexus/var/cache/nexus/.ekey-cache",
    "rb"
).read()

assert data[:8] == b"NXKEY_V1"

nonce = data[8:20]
ciphertext = data[20:]

plaintext = AESGCM(key).decrypt(
    nonce,
    ciphertext,
    None
)

print(plaintext.decode())
```

Running this produced the decrypted plaintext, which was the stolen emergency access key and contained the flag in `CSSCTF{...}` format.

## Final Forensic Conclusion

The central trick in this challenge was that the relay's own application log could not be trusted. It claimed:

```
03:17:03 principal=KAI-7
```

but every piece of lower level, harder to fake evidence told a different story:

```
KAI-7 session ended        -> 03:05:41
svc-relay session started  -> 03:10:02
nx-export --emergency      -> ran as UID 998 (svc-relay)
database job record        -> tied to svc-relay session
```

The SQLite database supplied the exact session UUID and event epoch needed for the key derivation, and `/etc/machine-id` supplied the machine identifier. Those three values together reproduce the SHA-256-derived AES-256 key required to decrypt `.ekey-cache` and recover the flag.

The overall solve path looked like this:

```
Filesystem
    |
    v
Logs
    |
    v
Relay log attribution to KAI-7 contradicted by auth log and audit log
    |
    v
SQLite session and job records (session UUID, event epoch)
    |
    v
machine-id + session UUID + event epoch
    |
    v
SHA-256
    |
    v
AES-256-GCM decryption
    |
    v
Emergency access key
    |
    v
CSSCTF{...}
```

## Key Takeaways

**Application logs can lie, intentionally or otherwise.** The relay log attributed the export to a user who had already logged out several minutes earlier. Trusting it at face value would have led to the wrong conclusion entirely.

**Lower level evidence is harder to fake.** The audit log records the actual UID a process ran as at the kernel level, which is a much stronger signal than an application choosing what name to write into its own log file.

**Cross referencing multiple sources breaks a false narrative.** It took the auth log, the audit log, and the SQLite database all pointing the same direction to firmly establish that `svc-relay`, not Kai, performed the export.

**Reading the actual source code removes all guesswork.** Once `exporter.py` was found, there was no need to guess at the key derivation scheme or the file format, both were spelled out directly in the code.
