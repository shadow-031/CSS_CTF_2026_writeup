# Echoes of the Relay - CTF Writeup

**Category:** Forensics

**Flag:** `CSSCTF{d3l3t3d_d03snt_m34n_g0n3}`

## Challenge Overview

The challenge provides a single file, `relay_backup.img`, described as a damaged storage image taken from a maintenance terminal. The files that are immediately visible on the image are essentially useless, but the operator supposedly tried to preserve something before shutdown happened, which is the thread to pull on here.

## 1. Identify the Image

The first step with any unknown image file is just figuring out what it actually is.

```bash
file relay_backup.img
```

This identified it as a 32 MB ext4 filesystem, with the volume label `KUIPER_RELAY`. Since it was already a plain ext4 image rather than something wrapped in a partition table, there was no offset math needed before working with it directly.

## 2. Check the Visible Files

Rather than mounting the image, I used `debugfs` to pull out everything that was currently visible in the filesystem:

```bash
debugfs -R "rdump / ./ex" relay_backup.img
```

Reading through what came out, two files stood out. The shift report flatly stated "Nothing unusual detected", which read like a deliberate red herring given the challenge framing. More interesting was `diagnostics.txt`, which warned of "Residual filesystem entries detected". That phrase is a pretty direct hint toward deleted files still having recoverable traces on the filesystem, which is exactly the kind of thing ext4 is known for.

## 3. Recover the Deleted Inode

Ext4 does not securely wipe data when a file is deleted, it just removes the directory entry and marks the inode as free. If nothing has overwritten that space yet, the content is often still sitting there. `debugfs` has a built in way to check for this:

```bash
debugfs -R "lsdel" relay_backup.img
```

This listed exactly one deleted inode, inode #25, 303 bytes in size. Since the size was small and the entry was still listed, I just read it directly:

```bash
debugfs -R "cat <25>" relay_backup.img
```

The recovered content turned out to be a personal note from an operator identified as NX-17:

```
I attached the recovery package to the diagnostic image before the node went offline.
archive password = severance2101
Look beyond what the image viewer shows you.
```

This single deleted file handed over almost everything needed for the rest of the challenge: a password, and a strong hint that something is hidden inside an image file specifically.

## 4. Find the Hidden Archive

The phrase "diagnostic image" pointed toward `operator/Pictures/relay_status.png`, which was 19,834 bytes. A PNG file has a well defined structure that ends with an `IEND` chunk, and anything after that chunk is not part of the actual image data as far as any PNG decoder is concerned, which is exactly why "look beyond what the image viewer shows you" was such a pointed clue.

Checking where the real PNG data ends versus where the file actually ends, the `IEND` chunk finished at byte 19,274, but the file itself continued on to 19,834 bytes. Looking at what came after that point, the trailing bytes started with:

```
PK\x03\x04
```

which is the standard local file header magic for a ZIP archive. So a ZIP file had simply been appended directly onto the end of a valid PNG, something that works perfectly fine because image viewers only read up through `IEND` and never notice the extra bytes tacked on afterward.

Extracting just that trailing portion:

```python
d = open('relay_status.png', 'rb').read()
open('z.zip', 'wb').write(d[19274:])
```

## 5. Extract With the Password

With the ZIP isolated and the password already recovered from the deleted inode note, extraction was straightforward:

```bash
unzip -P severance2101 z.zip
```

The archive contained two files:

```
transmission/manifest.txt
transmission/core_recovery.txt
```

The second file held the flag.

## Final Flag

```
CSSCTF{d3l3t3d_d03snt_m34n_g0n3}
```

## Key Takeaways

**Deleting a file on ext4 does not erase its contents.** It only removes the directory entry and frees the inode for reuse. Until something overwrites that space, tools like `debugfs lsdel` and `debugfs cat <inode>` can recover the data directly.

**Appending data after a PNG's `IEND` chunk is invisible to normal viewers.** Since image viewers and decoders stop reading once they hit the end of image marker, a file can look completely normal while quietly carrying extra data, in this case a whole ZIP archive, tacked onto the end. Checking for unexpected magic bytes past the expected end of file, or just running something like `binwalk`, catches this immediately.

**The in-game text was the real clue chain.** "Residual filesystem entries detected" pointed straight at the deleted file, and "look beyond what the image viewer shows you" pointed straight at the appended archive. Neither clue was decorative, both described the exact technique needed.
