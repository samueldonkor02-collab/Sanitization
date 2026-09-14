# Sanitization
Simulated a disk wipe on a virtual drive, recovered the deleted files with TestDisk, then permanently destroyed sensitive data with shred — a hands-on look at both sides of media sanitization
Media Sanitization Lab (TestDisk Recovery vs. Secure Deletion with Shred)

TestDisk | Kali Linux | CompTIA Labs | fdisk | dd | mkfs | shred | Media Sanitization

================================================================================
OVERVIEW
================================================================================
This lab looks at data sanitization from both directions. First I recovered
files off a drive after its partition table and filesystem got wiped, then
I went the other way and used shred to make sure specific files couldn't
come back the same way. Everything ran against a secondary virtual disk in
a CompTIA guided lab (Kali Linux, hosted through labclient.labondemand.com),
standing in for a company "SalesStorage" drive.

The point wasn't just "run TestDisk, get files back." It was understanding
why a wipe with dd still leaves data recoverable, and why shred is what
you'd actually reach for if the goal is to make sure something is gone for
good, not just deleted.

================================================================================
OBJECTIVE
================================================================================
Build a partition, put representative data on it, simulate a disk
wipe/failure, recover what I could with TestDisk, and then contrast that
against files that had been properly shredded so TestDisk had nothing left
to find.

================================================================================
ENVIRONMENT
================================================================================
- Working box: Kali Linux VM, root shell
- Target disk: /dev/sdb, 80 GiB (85,899,345,920 bytes / 167,772,160 sectors),
  shown as "Msft Virtual Disk"
- Mount point: /mnt/SalesStorage
- Lab platform: CompTIA Learning Platform, hosted lab via LabClient
  (labclient.labondemand.com)

================================================================================
TOOLS I USED
================================================================================
fdisk         - Created and inspected the MBR partition table on /dev/sdb
mkfs.fat      - Formatted the new partition as FAT32
dd            - Wiped the raw disk to simulate a destroyed partition table
TestDisk      - Located and recovered the FAT32 partition and directory
                tree after the wipe
shred         - Overwrote and destroyed specific files/directories so they
                couldn't be recovered the same way
find          - Batched shred across every file in a target directory
mount/umount  - Attached and detached /dev/sdb1 at /mnt/SalesStorage
cp            - Populated the drive and copied recovered data back out

================================================================================
WHAT I DID
================================================================================

1. Setting up the SalesStorage partition
   I started by partitioning /dev/sdb. Ran `fdisk /dev/sdb`, after a typo
   (`fdisk/dev/sdb`, missing the space) that just gave a clean zsh "no such
   file or directory" error instead of doing anything harmful.

   fdisk reported the disk had no recognized partition table and created a
   new DOS (MBR) disklabel (identifier 0xa947f419). From there I used `n`
   -> `p` -> partition 1 -> accepted the default start/end sectors, which
   gave me an 80 GiB Linux partition (/dev/sdb1, 167,770,112 sectors), then
   wrote the table.

   Formatted it with `mkfs -t fat /dev/sdb1`, made a mount point with
   `mkdir /mnt/SalesStorage`, and mounted it. Then populated the drive with
   SecLists as a stand-in for real business data:
       cp -r /usr/share/seclists/* /mnt/SalesStorage/

   That gave me a realistic-looking structure to work with: Discovery,
   Fuzzing, IOCs, Miscellaneous, Passwords (with a Software subfolder
   holding files like cain-and-abel.txt and john-the-ripper.txt, and a
   WiFi-WPA subfolder with probable-v2-wpa-topN wordlists), Pattern-
   Matching, Payloads, Usernames, Web-Shells, and a README.md. Confirmed
   the copy landed with `ls -l /mnt/SalesStorage`.

2. Simulating a disk wipe
   To simulate a real failure, I ran:
       dd if=/dev/zero of=/dev/sdb bs=1M status=progress
   That overwrote the start of the raw disk and killed the partition table
   and filesystem metadata (roughly 86 GB copied at about 1.2 GB/s).

   Running `fdisk /dev/sdb` again confirmed the damage. It warned the disk
   was in use and that repartitioning was risky, then reported it no
   longer contained a recognized partition table, auto-creating a fresh
   DOS (MBR) disklabel (identifier 0x45eff666) just from being opened.

   Tried reading a file directly with
   `cat Downloads/Passwords/nt4-password.txt` and got "No such file or
   directory," which was expected since the normal filesystem path was
   gone. Unmounted the broken volume with `umount /mnt/SalesStorage`.

3. Recovering with TestDisk
   Ran TestDisk against the disk. The very first read flagged the problem
   directly: "Partition sector doesn't have the endmark 0xAA55."

   Selected the Intel/PC partition table type and ran TestDisk's Quick
   Search / "Try to locate partition." Against the freshly-zeroed area,
   TestDisk came back with "No file found, filesystem may be damaged,"
   which made sense since that portion of the FAT32 structures really had
   been overwritten.

   Ran TestDisk again, this time directly against the surviving partition
   (/dev/sdb1, 85 GB / 79 GiB), and this time it found an intact FAT32
   volume with the original directory tree still there: Discovery,
   Fuzzing, IOCs, Miscellaneous, Passwords, Pattern-Matching, Payloads,
   README.md, Usernames, and Web-Shells.

   Selected the directory and used TestDisk's `C` (copy selected files)
   option. The log confirmed it: "Copy done! 177 ok, 0 failed." Remounted
   /dev/sdb1 to /mnt/SalesStorage and checked the recovered contents with
   `ls -l`.

4. Recoverable deletion vs. actual sanitization
   To see the other side of this, I picked one sensitive file (renamed
   "BiblePass," sitting in Passwords) and ran shred directly against it.
   Watching it work was the interesting part: shred renamed the file
   through several obfuscated names before finally deleting it (down
   through 000000 -> 00000 -> ... -> BiblePass_part01.txt,
   _part02.txt, and so on), running multiple randomized overwrite passes
   along the way ("pass 1/4 (random)...", "pass 2/4 (random)...").

   Then ran a bulk pass across the whole recovered Passwords directory,
   once it had served its purpose in the lab:
       find /mnt/SalesStorage/Passwords -type f -exec shred -uvz {} \;

   Hit a mistake mid-run here: shred kept throwing "invalid option -- '/'"
   errors. That was a sign the command was malformed somewhere, a stray
   character was getting read as a flag instead of part of a path, which
   was a good reminder to slow down and check quoting on `find -exec`
   constructions rather than assume the command had just quietly worked.

   Cleaned up the leftover non-essential recovered data with
   `rm -r /mnt/SalesStorage/Miscellaneous` and confirmed the final state
   with `ls -l /mnt/SalesStorage`.

================================================================================
WHAT'S IN THIS REPO
================================================================================
media-sanitization-lab/
|-- README.txt
`-- screenshots/
    |-- 01-fdisk-new-mbr-label.png
    |-- 02-fdisk-partition-created.png
    |-- 03-mkfs-fat-format.png
    |-- 04-seclists-copy-populate.png
    |-- 05-dd-wipe-progress.png
    |-- 06-fdisk-post-wipe-no-table.png
    |-- 07-testdisk-partition-endmark-error.png
    |-- 08-testdisk-no-file-found.png
    |-- 09-testdisk-recovered-directory-listing.png
    |-- 10-testdisk-copy-done.png
    |-- 11-shred-biblepass-rename-passes.png
    |-- 12-shred-bulk-passwords-directory.png
    `-- 13-shred-invalid-option-error.png

================================================================================
SKILLS I PICKED UP
================================================================================
- Reading partition table corruption directly off the error messages (a
  missing 0xAA55 endmark, "device does not contain a recognized partition
  table") instead of assuming a wipe automatically means everything's
  gone for good.
- Seeing exactly how dd's raw overwrite kills a partition table and
  filesystem metadata while leaving a lot of the underlying data blocks
  untouched, at least at first, which is exactly why TestDisk-style
  recovery is possible in the first place.
- Working through TestDisk step by step: picking the right partition
  table type, running Quick Search, telling a still-readable filesystem
  apart from one that's actually been overwritten, and using the built-in
  copy function to pull back a whole directory tree in one go.
- Understanding shred as basically the opposite of what TestDisk relies
  on, multiple randomized overwrite passes plus filename obfuscation, on
  purpose, to remove the recoverability that a plain rm or an accidental
  wipe doesn't guarantee.
- Debugging a live shell mistake instead of just moving past it, tracking
  down a repeated "invalid option -- '/'" error back to a malformed
  shred/find command rather than assuming it had just failed for no
  reason.

================================================================================
HOW THIS APPLIES IN THE REAL WORLD
================================================================================
This is really the flip side of incident response. Not every "we lost the
drive" situation is actually unrecoverable, and not every deleted file
should be treated as gone. Knowing the difference between a corrupted
partition table, data that's genuinely been overwritten, and data that's
been properly shredded matters for two roles that often land on the same
desk: recovering from accidental wipes or early-stage ransomware, and
making sure sensitive data is actually destroyed, not just deleted,
before a drive gets reused or thrown out.

================================================================================
WHERE I'M COMING FROM
================================================================================
I'm making the jump into cybersecurity from a background in healthcare.
It's a different field on paper, but a lot of the muscle memory carries
over: following procedures carefully, protecting sensitive information,
staying calm and methodical when something isn't working the way it's
supposed to. I'm currently studying for CompTIA Security+ and building
labs like this one to get real hands-on reps in, since that's what I'm
missing on paper right now compared to my experience.

================================================================================
WHAT I WANT TO LEARN NEXT
================================================================================
- Practicing recovery against more damaged filesystems, partial
  overwrites, and non-FAT filesystems
- Comparing shred against other sanitization methods, like ATA Secure
  Erase or cryptographic erase, since shred's overwrite model doesn't
  fully hold up on SSDs with wear leveling and TRIM
- Building out a documented "recover, then sanitize" workflow that could
  double as a real chain-of-custody process

================================================================================
LIMITATIONS & WHAT I'D DO DIFFERENTLY IN PRODUCTION
================================================================================
- This lab used a FAT32 filesystem on a virtual disk. Recovery behavior
  and shred's effectiveness look very different on SSDs with wear
  leveling and TRIM, where overwrite-based tools like shred aren't
  guaranteed to reach every physical copy of the data.
- A real recovery engagement would work from a forensic image of the
  original media instead of operating directly on the live device, so
  there's no risk of overwriting data that could still have been
  recovered.
- Sanitization in production should follow a documented standard, like
  NIST SP 800-88, rather than an ad hoc shred pass.

================================================================================
REFERENCES
================================================================================
- TestDisk documentation: https://www.cgsecurity.org/wiki/TestDisk
- GNU coreutils shred man page
- NIST SP 800-88, Guidelines for Media Sanitization
- CompTIA Security+ (SY0-701) Exam Objectives:
  https://www.comptia.org/certifications/security
- Kali Linux, attacker VM environment used throughout
