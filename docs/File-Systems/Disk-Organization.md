# Disk Organization

## Introduction

Disk organization refers to the way data is physically and logically arranged on storage devices—how disks are structured at the hardware level, how they are divided into usable regions, and how the operating system views and manages this storage. Before a file system can organize files, the underlying disk must be organized into a form the file system can use.

This document explores disk organization from the physical hardware level (platters, tracks, sectors) through logical structures (partitions, volumes, block devices) to the abstractions the operating system presents to file systems. Understanding disk organization is essential for understanding why file systems are designed the way they are and how performance characteristics emerge from physical reality.

---

## Physical Disk Structure

### Hard Disk Drive (HDD) Anatomy

A traditional hard disk drive stores data magnetically on rotating platters:

```
HDD Physical Structure:

        ┌─────────────────────────────────────────┐
        │                                          │
        │           ┌─────────────┐                │
        │           │   Spindle   │                │
        │           └──────┬──────┘                │
        │                  │                       │
        │     ┌────────────┼────────────┐          │
        │     │            │            │          │
        │  ┌──┴──┐      ┌──┴──┐      ┌──┴──┐      │
        │  │Platter│    │Platter│    │Platter│    │
        │  │  1   │     │  2   │     │  3   │     │
        │  └──┬──┘      └──┬──┘      └──┬──┘      │
        │     │            │            │          │
        │     └────────────┼────────────┘          │
        │                  │                       │
        │           ┌──────┴──────┐                │
        │           │  Actuator   │                │
        │           │    Arm      │                │
        │           └──────┬──────┘                │
        │                  │                       │
        │           ┌──────┴──────┐                │
        │           │  Read/Write │                │
        │           │    Head     │                │
        │           └─────────────┘                │
        │                                          │
        └─────────────────────────────────────────┘
```

### Key Physical Components

| Component                | Description                                               | Relevance                                  |
|--------------------------|-----------------------------------------------------------|--------------------------------------------|
| Platter                  | Circular magnetic disk                                     | Stores data magnetically                   |
| Spindle                  | Central axis                                              | Rotates platters at constant speed        |
| Track                    | Concentric circle on platter                              | Where data is written                      |
| Sector                   | Subdivision of track                                      | Smallest addressable unit (traditionally 512 bytes) |
| Cylinder                 | Same track across all platters                            | Reduces head movement                      |
| Read/Write Head          | Reads/writes magnetic data                                | One per platter surface                    |
| Actuator Arm             | Moves heads radially                                      | Positions heads over tracks                |

### Disk Geometry

```
Disk Geometry Visualization:

Top view of a platter:

         Track 0 (outermost)
        ╭──────────────────╮
       ╱   Track 1          ╲
      │   ╭──────────────╮   │
      │  │   Track 2      │  │
      │  │  ╭──────────╮  │  │
      │  │  │  Track 3 │  │  │
      │  │  │  ╭────╮  │  │  │
      │  │  │  │Spindle│ │  │  │
      │  │  │  ╰────╯  │  │  │
      │  │  ╰──────────╯  │  │
      │  ╰──────────────╯   │
       ╲                    ╱
        ╰──────────────────╯

Sector within a track:

    Track 2
    ╭─────────────────────────────────╮
    │ [S0][S1][S2][S3][S4][S5][S6]... │
    ╰─────────────────────────────────╯
    Each sector: typically 512 bytes or 4 KB

Cylinder concept:
    Cylinder N = Track N on platter 0
               + Track N on platter 1
               + Track N on platter 2
               + ...

    All tracks at the same radial position
```

### Disk Addressing

Historically, disks used CHS (Cylinder-Head-Sector) addressing:

```
CHS Addressing:

Address = (Cylinder, Head, Sector)

├── Cylinder: Radial position (0 to max)
├── Head: Which platter surface (0 to max)
├── Sector: Position within track (1 to max, traditionally 1-based)
└── Total addressable sectors = Cylinders × Heads × Sectors

Example:
├── 1024 cylinders
├── 256 heads
├── 63 sectors/track
├── Sector size: 512 bytes
└── Total capacity: 1024 × 256 × 63 × 512 = 8.4 GB
```

Modern disks use LBA (Logical Block Addressing):

```
LBA Addressing:

Address = Linear block number (0 to max)

├── Simple linear addressing
├── OS doesn't need to know geometry
├── Disk controller translates LBA to physical location
├── LBA 0 = first sector
├── LBA N = (N+1)th sector
└── Much simpler for software

Advantages:
├── No geometry limitations
├── Supports any disk size
├── Simpler programming model
├── Better for SSDs (no geometry)
└── Universal standard since late 1990s
```

### Solid-State Drive (SSD) Organization

SSDs have fundamentally different organization:

```
SSD Structure:

        ┌─────────────────────────────────────┐
        │            SSD Controller            │
        │  ┌─────────────────────────────────┐ │
        │  │  Flash Translation Layer (FTL)  │ │
        │  │  - LBA → Physical mapping       │ │
        │  │  - Wear leveling                │ │
        │  │  - Garbage collection           │ │
        │  └─────────────────────────────────┘ │
        └───────────────┬─────────────────────┘
                        │
        ┌───────────────┼─────────────────────┐
        │               │                     │
   ┌────┴────┐    ┌────┴────┐         ┌────┴────┐
   │ NAND    │    │ NAND    │         │ NAND    │
   │ Chip 1  │    │ Chip 2  │   ...   │ Chip N  │
   └────┬────┘    └────┬────┘         └────┬────┘
        │              │                    │
   ┌────┴────┐    ┌────┴────┐         ┌────┴────┐
   │  Dies   │    │  Dies   │         │  Dies   │
   └────┬────┘    └────┬────┘         └────┬────┘
        │              │                    │
   ┌────┴────┐    ┌────┴────┐         ┌────┴────┐
   │ Planes  │    │ Planes  │         │ Planes  │
   └────┬────┘    └────┬────┘         └────┬────┘
        │              │                    │
   ┌────┴────┐    ┌────┴────┐         ┌────┴────┐
   │ Blocks  │    │ Blocks  │         │ Blocks  │
   └────┬────┘    └────┬────┘         └────┬────┘
        │              │                    │
   ┌────┴────┐    ┌────┴────┐         ┌────┴────┐
   │  Pages  │    │  Pages  │         │  Pages  │
   └─────────┘    └─────────┘         └─────────┘

Key differences from HDD:
├── No moving parts (no seek time)
├── Random access as fast as sequential
├── Must erase entire blocks before writing
├── Limited write cycles per block
├── Wear leveling required
└── FTL hides complexity from OS
```

### SSD vs. HDD Characteristics

| Characteristic          | HDD                | SSD               |
|-------------------------|--------------------|--------------------|
| Access Time             | 5-15 ms            | 0.05-0.1 ms        |
| Sequential Read         | 100-200 MB/s       | 500-7000 MB/s      |
| Random Read (IOPS)      | 100-200            | 100,000-1,000,000  |
| Random Write (IOPS)     | 100-200            | 50,000-500,000     |
| Noise                   | Audible            | Silent             |
| Power Consumption       | 5-10 W             | 2-5 W              |
| Shock Resistance        | Low                | High               |
| Cost per GB            | Low                | High               |
| Lifespan                | Mechanical failure  | Write cycle limit   |
| Fragmentation Impact    | High               | Minimal            |

---

## Disk Performance Characteristics

### Access Time Components (HDD)

```
HDD Access Time:

Access Time = Seek Time + Rotational Latency + Transfer Time

1. Seek Time:
   ├── Time to move read/write head to correct track
   ├── Average: 4-10 ms
   ├── Depends on distance traveled
   ├── Dominant factor for random access
   └── Reduced by limiting head movement

2. Rotational Latency:
   ├── Time for sector to rotate under head
   ├── Average: ½ rotation time
   ├── 7200 RPM → 4.17 ms average
   ├── 5400 RPM → 5.56 ms average
   ├── 15000 RPM → 2 ms average
   └── Depends on rotational speed

3. Transfer Time:
   ├── Time to read/write actual data
   ├── Depends on data size
   ├── Sequential: faster (no seeks)
   ├── Depends on interface (SATA, SAS, NVMe)
   └── Typically 100-200 MB/s for HDD

Total average access time:
├── Random: ~8-15 ms (seek + rotation)
├── Sequential: dominated by transfer rate
└── Consecutive blocks much faster
```

### Why Sequential Access Matters

```
Sequential vs. Random Access (HDD):

Reading 1000 blocks of 4 KB each:

Sequential (contiguous):
├── Seek time: 1 seek (8 ms)
├── Rotational latency: 1 rotation (4 ms)
├── Transfer: 1000 × 4 KB = 4 MB at 150 MB/s = 27 ms
├── Total: ~39 ms
└── Effective rate: ~102 MB/s

Random (scattered):
├── Seek time: 1000 seeks × 8 ms = 8000 ms
├── Rotational latency: 1000 × 4 ms = 4000 ms
├── Transfer: 4 MB at 150 MB/s = 27 ms
├── Total: ~12,027 ms
└── Effective rate: ~0.33 MB/s

Sequential is ~300x faster!
This is why file allocation strategy matters.
```

### SSD Performance Characteristics

```
SSD Performance:

No mechanical components:
├── No seek time
├── No rotational latency
├── Random access ≈ sequential access
├── Latency: ~0.1 ms
└── IOPS: 100,000-1,000,000

But different constraints:
├── Erase-before-write:
│   ├── Can write pages (4 KB) individually
│   ├── Must erase blocks (256 KB - 4 MB) to rewrite
│   ├── Erasing is slow
│   ├── Write amplification possible
│   └── FTL manages this transparently
│
├── Limited write cycles:
│   ├── Each block has limited P/E cycles
│   ├── TLC NAND: ~1000-3000 cycles
│   ├── MLC NAND: ~3000-10000 cycles
│   ├── SLC NAND: ~100,000 cycles
│   └── Wear leveling spreads writes
│
├── Garbage collection:
│   ├── Must move valid pages before erasing
│   ├── Causes background activity
│   ├── Can cause latency spikes
│   └── TRIM helps identify free blocks
│
└── Over-provisioning:
    ├── Extra capacity for management
    ├── Typically 7-28% of total
    ├── User can't access this space
    └── Used for wear leveling, GC
```

---

## Disk Partitioning

### The Partition Concept

A partition is a contiguous region of a disk treated as a separate logical device:

```
Disk Partitioning:

Physical Disk (1 TB):
┌───────────────────────────────────────────────────────┐
│                                                       │
│  ┌─────────┬────────────┬──────────────────────────┐ │
│  │Partition│ Partition  │       Partition          │ │
│  │   1     │     2      │          3               │ │
│  │ (100 GB)│  (200 GB)  │       (700 GB)           │ │
│  └─────────┴────────────┴──────────────────────────┘ │
│                                                       │
└───────────────────────────────────────────────────────┘

Each partition:
├── Acts as independent disk
├── Can have its own file system
├── Can be formatted independently
├── Can be resized (with tools)
└── Appears as separate device (/dev/sda1, /dev/sda2, etc.)

Why partition?
├── Separate OS and data
├── Multiple operating systems
├── Different file systems for different needs
├── Isolation of failures
├── Easier backup and management
└── Security boundaries
```

### Partition Table Types

Two main partition table formats exist:

```
MBR (Master Boot Record):

Location: First 512 bytes of disk (LBA 0)
├── Boot code: 446 bytes
├── Partition entries: 4 × 16 bytes = 64 bytes
├── Signature: 2 bytes (0x55AA)
└── Total: 512 bytes

Limitations:
├── Maximum 4 primary partitions
├── Maximum 2 TB disk size
├── Uses 32-bit LBA addresses
├── Extended partitions for >4 partitions
└── Legacy standard (still common)

Structure:
┌──────────────────────────────────────────────┐
│                 MBR (512 bytes)              │
├────────────┬────────────┬────────────┬───────┤
│ Boot Code  │ Partition  │ Partition  │ ...   │
│ (446 B)    │ Entry 1    │ Entry 2    │       │
│            │ (16 B)     │ (16 B)     │       │
├────────────┴────────────┴────────────┴───────┤
│              Signature (0x55AA)              │
└──────────────────────────────────────────────┘

GPT (GUID Partition Table):

Location: LBA 1 (after protective MBR)
├── Primary GPT header
├── Partition entries
├── Backup GPT at end of disk
├── CRC32 checksums for integrity
└── Modern standard

Advantages:
├── Up to 128 partitions (typically)
├── Supports disks up to 9.4 ZB
├── 64-bit LBA addresses
├── Redundant partition table
├── CRC protection
├── Unique partition GUIDs
└── Required for UEFI boot

Structure:
┌──────────────────────────────────────────────┐
│  Protective MBR (LBA 0)                      │
├──────────────────────────────────────────────┤
│  GPT Header (LBA 1)                          │
├──────────────────────────────────────────────┤
│  Partition Entries (LBA 2-33)                │
├──────────────────────────────────────────────┤
│                                              │
│  Partition Data                              │
│                                              │
├──────────────────────────────────────────────┤
│  Backup Partition Entries                    │
├──────────────────────────────────────────────┤
│  Backup GPT Header (last LBA)                │
└──────────────────────────────────────────────┘
```

### Partition Types

Partitions have type identifiers that indicate their intended use:

```
Common Partition Types (GPT GUIDs):

Linux:
├── Linux filesystem: 0FC63DAF-8483-4772-8E79-3D69D8477DE4
├── Linux swap: 0657FD6D-A4AB-43C4-84E5-0933C84B4F4F
├── Linux LVM: E6D6D379-F507-44C2-A23C-238F2A3DF928
└── Linux RAID: A19D880F-05FC-4D3B-A006-743F0F84911E

Windows:
├── EFI System: C12A7328-F81F-11D2-BA4B-00A0C93EC93B
├── Microsoft Reserved: E3C9E316-0B5C-4DB8-817D-F92DF00215AE
├── Basic Data: EBD0A0A2-B9E5-4433-87C0-68B6B72699C7
└── Recovery: DE94BBA4-06D1-4D40-A16A-BFD50179D6AC

Apple:
├── Apple HFS+: 48465300-0000-11AA-AA11-00306543ECAC
├── Apple APFS: 7C3457EF-0000-11AA-AA11-00306543ECAC
└── Apple Boot: 426F6F74-0000-11AA-AA11-00306543ECAC

Other:
├── BIOS Boot: 21686148-6449-6E6F-744E-656564454649
├── Intel Fast Flash: D3BFE2DE-3DAF-11DF-BA40-E3A556D89593
└── Sony Boot: F4019732-066E-4E12-8273-346C5641494F
```

### Partitioning Schemes

```
Partitioning Examples:

Simple Desktop (UEFI):
├── /dev/nvme0n1p1: EFI System (512 MB, FAT32)
├── /dev/nvme0n1p2: Linux root (100 GB, ext4)
└── /dev/nvme0n1p3: Linux home (rest, ext4)

Dual-Boot (Windows + Linux):
├── /dev/sda1: EFI System (512 MB, FAT32)
├── /dev/sda2: Windows Reserved (16 MB)
├── /dev/sda3: Windows C: (200 GB, NTFS)
├── /dev/sda4: Windows Recovery (500 MB)
├── /dev/sda5: Linux root (100 GB, ext4)
├── /dev/sda6: Linux home (300 GB, ext4)
└── /dev/sda7: Linux swap (16 GB)

Server (Separate Partitions):
├── /dev/sda1: /boot (1 GB, ext4)
├── /dev/sda2: / (50 GB, ext4)
├── /dev/sda3: /var (100 GB, ext4)
├── /dev/sda4: /home (500 GB, ext4)
├── /dev/sda5: /tmp (20 GB, ext4)
└── /dev/sda6: swap (64 GB)

Database Server:
├── /dev/sda1: OS (100 GB, ext4)
├── /dev/sdb1: Database data (LVM, XFS)
└── /dev/sdc1: Database logs (LVM, XFS)
    (Separate disks for performance)
```

---

## Logical Volume Management (LVM)

LVM provides a layer of abstraction between physical disks and file systems:

### LVM Concepts

```
LVM Architecture:

Physical Layer:
┌─────────┐  ┌─────────┐  ┌─────────┐
│ /dev/sda│  │ /dev/sdb│  │ /dev/sdc│
│  (1 TB) │  │  (2 TB) │  │  (500GB)│
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     ▼            ▼            ▼
Physical Volumes (PVs):
┌─────────┐  ┌─────────┐  ┌─────────┐
│  PV1    │  │  PV2    │  │  PV3    │
│ (1 TB)  │  │ (2 TB)  │  │ (500GB) │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┼────────────┘
                  │
                  ▼
Volume Group (VG):
┌─────────────────────────────────────┐
│         VG "data" (3.5 TB)          │
└─────────────────┬───────────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
     ▼            ▼            ▼
Logical Volumes (LVs):
┌─────────┐  ┌─────────┐  ┌─────────┐
│  LV1    │  │  LV2    │  │  LV3    │
│ (500GB) │  │ (1 TB)  │  │ (2 TB)  │
│ ext4    │  │ XFS     │  │ ext4    │
└─────────┘  └─────────┘  └─────────┘
   /home       /data        /backup
```

### LVM Advantages

```
LVM Benefits:

1. Flexible Resizing
   ├── Grow LV without unmounting (usually)
   ├── Shrink LV (requires unmount)
   ├── Reallocate space between LVs
   └── No repartitioning needed

2. Snapshots
   ├── Point-in-time copies
   ├── Instant creation
   ├── Space-efficient (CoW)
   └── Perfect for backups

3. Spanning
   ├── LV can span multiple physical disks
   ├── Combine smaller disks into large volume
   ├── Transparent to file system
   └── Example: 3 × 1 TB = 1 × 3 TB LV

4. Striping
   ├── Distribute data across multiple disks
   ├── Improves performance
   ├── Similar to RAID 0
   └── Configurable stripe size

5. Mirroring
   ├── Replicate data across disks
   ├── Similar to RAID 1
   ├── Protects against disk failure
   └── Can combine with striping

6. Thin Provisioning
   ├── Allocate space on demand
   ├── Overcommit storage
   ├── Efficient use of space
   └── Snapshots more efficient
```

### LVM Commands and Workflow

```
LVM Workflow:

1. Create Physical Volumes:
   $ pvcreate /dev/sda /dev/sdb
   
2. Create Volume Group:
   $ vgcreate data /dev/sda /dev/sdb
   
3. Create Logical Volumes:
   $ lvcreate -L 500G -n home data
   $ lvcreate -L 1T -n storage data
   
4. Create File Systems:
   $ mkfs.ext4 /dev/data/home
   $ mkfs.xfs /dev/data/storage
   
5. Mount and Use:
   $ mount /dev/data/home /home
   $ mount /dev/data/storage /storage

Common Operations:
├── Extend LV: lvextend -L +100G /dev/data/home
├── Extend FS: resize2fs /dev/data/home
├── Create Snapshot: lvcreate -L 10G -s -n snap /dev/data/home
├── Add Disk to VG: vgextend data /dev/sdc
└── Remove Disk: pvmove /dev/sda (move data first)
```

---

## RAID (Redundant Array of Independent Disks)

RAID combines multiple disks for performance, redundancy, or both:

### RAID Levels

```
RAID 0 (Striping):
┌─────┐ ┌─────┐ ┌─────┐
│Disk1│ │Disk2│ │Disk3│
│ A1  │ │ A2  │ │ A3  │
│ B1  │ │ B2  │ │ B3  │
│ C1  │ │ C2  │ │ C3  │
└─────┘ └─────┘ └─────┘

├── Data split across disks
├── No redundancy
├── Full capacity usable
├── Fastest performance
├── One disk fails → all data lost
└── Use: Temporary/scratch data

RAID 1 (Mirroring):
┌─────┐ ┌─────┐
│Disk1│ │Disk2│
│ A   │ │ A   │
│ B   │ │ B   │
│ C   │ │ C   │
└─────┘ └─────┘

├── Data duplicated on both disks
├── Can survive one disk failure
├── 50% capacity usable
├── Read performance improved
├── Write performance similar
└── Use: OS disks, critical data

RAID 5 (Striping with Parity):
┌─────┐ ┌─────┐ ┌─────┐
│Disk1│ │Disk2│ │Disk3│
│ A1  │ │ A2  │ │ Ap  │
│ Bp  │ │ B1  │ │ B2  │
│ C2  │ │ Cp  │ │ C1  │
└─────┘ └─────┘ └─────┘

├── Data + distributed parity
├── Can survive one disk failure
├── (n-1)/n capacity usable
├── Good read performance
├── Slower writes (parity calculation)
└── Use: General storage, file servers

RAID 6 (Double Parity):
┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐
│Disk1│ │Disk2│ │Disk3│ │Disk4│
│ A1  │ │ A2  │ │ Ap  │ │ Aq  │
│ Bp  │ │ B1  │ │ B2  │ │ Bq  │
│ Cq  │ │ Cp  │ │ C1  │ │ C2  │
└─────┘ └─────┘ └─────┘ └─────┘

├── Two parity blocks per stripe
├── Can survive two disk failures
├── (n-2)/n capacity usable
├── Slower writes than RAID 5
└── Use: Large arrays, high reliability

RAID 10 (1+0, Mirrored Stripes):
┌─────┐ ┌─────┐   ┌─────┐ ┌─────┐
│Disk1│ │Disk2│   │Disk3│ │Disk4│
│ A1  │ │ A1  │   │ A2  │ │ A2  │
│ B1  │ │ B1  │   │ B2  │ │ B2  │
└─────┘ └─────┘   └─────┘ └─────┘
    Mirror            Mirror
    ←── Stripe ──→

├── Mirrored pairs, then striped
├── Can survive multiple failures
├── 50% capacity usable
├── Excellent performance
├── Fast rebuild
└── Use: Databases, high-performance
```

### RAID Comparison

| RAID  | Min Disks | Capacity      | Redundancy         | Read         | Write        |
|-------|-----------|---------------|---------------------|--------------|--------------|
| 0     | 2         | 100%          | None                | Excellent     | Excellent     |
| 1     | 2         | 50%           | 1 disk              | Good          | Moderate      |
| 5     | 3         | (n-1)/n       | 1 disk              | Good          | Moderate      |
| 6     | 4         | (n-2)/n       | 2 disks             | Good          | Poor          |
| 10    | 4         | 50%           | 1+ per mirror       | Excellent     | Good          |

### Hardware vs. Software RAID

```
Hardware RAID:
├── Dedicated RAID controller
├── OS sees single logical disk
├── Battery-backed cache
├── Better performance
├── More expensive
├── Controller-specific
└── Example: Dell PERC, LSI MegaRAID

Software RAID:
├── OS manages RAID
├── Uses CPU for parity
├── More flexible
├── Less expensive
├── Portable between systems
└── Examples: Linux mdraid, ZFS, Btrfs

Fake RAID:
├── Motherboard RAID
├── Requires driver
├── Not true hardware RAID
├── OS-dependent
└── Avoid if possible

Linux mdraid:
├── /dev/md0, /dev/md1, etc.
├── Created with mdadm
├── Supports all common RAID levels
├── Can be assembled at boot
└── Monitor with mdadm --monitor
```

---

## Block Devices and the OS

### Block Device Abstraction

The OS presents disks as block devices—devices that transfer data in fixed-size blocks:

```
Block Device Characteristics:

├── Fixed-size blocks (typically 512 B or 4 KB)
├── Random access (any block can be read)
├── Buffered by OS (page cache)
├── Can be mounted as file systems
├── Can be partitioned
└── Examples: /dev/sda, /dev/nvme0n1, /dev/vda

Character Devices (contrast):
├── Byte-by-byte access
├── Usually sequential
├── Not buffered by OS
├── Not mountable
└── Examples: /dev/tty, /dev/random
```

### Device Naming

```
Linux Device Naming:

SATA/SCSI/SAS:
├── /dev/sda - First disk
├── /dev/sdb - Second disk
├── /dev/sda1 - First partition on first disk
├── /dev/sda2 - Second partition
└── Naming can change between boots!

NVMe:
├── /dev/nvme0 - First NVMe disk
├── /dev/nvme0n1 - First namespace
├── /dev/nvme0n1p1 - First partition
└── Deterministic naming

Virtual:
├── /dev/vda - First virtio disk
├── /dev/xvda - Xen virtual disk
└── Depends on hypervisor

Stable Naming (udev):
├── /dev/disk/by-id/ - By hardware ID
├── /dev/disk/by-uuid/ - By filesystem UUID
├── /dev/disk/by-label/ - By filesystem label
├── /dev/disk/by-path/ - By physical path
└── Use these in /etc/fstab for reliability
```

### The Page Cache

The OS caches disk data in memory to improve performance:

```
Page Cache:

Memory:
┌─────────────────────────────────────────┐
│              Applications                │
├─────────────────────────────────────────┤
│              Page Cache                  │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐  │
│  │Page │Page │Page │Page │Page │Page │  │
│  │  1  │  2  │  3  │  4  │  5  │  6  │  │
│  └─────┴─────┴─────┴─────┴─────┴─────┘  │
├─────────────────────────────────────────┤
│              Disk Driver                 │
└─────────────────────────────────────────┘
                    │
                    ▼
              Physical Disk

Benefits:
├── Read caching: Recently read data in memory
├── Write buffering: Batch writes for efficiency
├── Read-ahead: Prefetch sequential data
├── Reduces disk I/O dramatically
└── Transparent to applications

Write Policies:
├── Write-through: Write to cache and disk immediately
├── Write-back: Write to cache, flush later
├── Write-back is faster but riskier
└── Linux uses write-back by default
```

---

## Disk Formatting

### Low-Level Formatting

```
Low-Level Formatting (Factory):

├── Done at factory
├── Creates physical sectors on disk
├── Writes sector headers, ECC data
├── Defines track/sector layout
├── Usually not done by users
└── Modern disks: done at manufacturing

Historically:
├── User could low-level format
├── Wrote sector markers, gaps
├── Checked for bad sectors
└── Obsolete for modern drives
```

### Partitioning

```
Partitioning:

├── Creates logical divisions on disk
├── Writes partition table (MBR or GPT)
├── Each partition is separate region
├── Does not write file system data
├── Non-destructive until formatting
└── Tools: fdisk, gdisk, parted, Disk Utility

Steps:
1. Identify disk: lsblk, fdisk -l
2. Create partition table: fdisk /dev/sda
3. Create partitions: n (new), t (type), w (write)
4. Verify: fdisk -l /dev/sda
```

### High-Level Formatting

```
High-Level Formatting (Creating File System):

├── Writes file system structures
├── Creates superblock, inodes, etc.
├── Initializes free space management
├── Prepares partition for file storage
├── Destroys existing data
└── Tools: mkfs.ext4, mkfs.xfs, mkfs.ntfs

Example:
$ mkfs.ext4 /dev/sda1
├── Creates ext4 file system
├── Writes superblock
├── Creates inode table
├── Initializes block bitmaps
└── Ready to mount

Journaling:
├── Journal area initialized
├── Consistent state guaranteed
├── Fast recovery after crash
└── Part of modern file systems
```

### Boot Blocks

```
Boot Process and Disk Layout:

MBR-based boot:
┌─────────────────────────────────────┐
│  MBR (LBA 0)                        │
│  ├── Boot code (446 bytes)          │
│  ├── Partition table (64 bytes)     │
│  └── Signature (2 bytes)            │
├─────────────────────────────────────┤
│  Partition 1 (bootable)             │
│  ├── Boot sector (VBR)              │
│  │   └── Loads bootloader           │
│  ├── Bootloader (GRUB stage 2)      │
│  └── File system                    │
├─────────────────────────────────────┤
│  Partition 2                        │
│  └── File system                    │
└─────────────────────────────────────┘

UEFI-based boot:
┌─────────────────────────────────────┐
│  Protective MBR (LBA 0)             │
├─────────────────────────────────────┤
│  GPT Header (LBA 1)                 │
├─────────────────────────────────────┤
│  GPT Entries (LBA 2-33)             │
├─────────────────────────────────────┤
│  EFI System Partition (ESP)         │
│  ├── FAT32 file system              │
│  ├── /EFI/BOOT/BOOTX64.EFI          │
│  ├── /EFI/ubuntu/grubx64.efi        │
│  └── Bootloaders and drivers        │
├─────────────────────────────────────┤
│  OS Partition                       │
│  └── File system                    │
└─────────────────────────────────────┘
```

---

## Modern Storage Technologies

### NVMe (Non-Volatile Memory Express)

```
NVMe:

├── Protocol for SSDs over PCIe
├── Much faster than SATA
├── Lower latency
├── Higher IOPS
├── Direct CPU connection
├── Up to 64K queues
├── 64K commands per queue
└── Designed for SSDs from scratch

Performance:
├── Sequential: 3,500-7,000 MB/s
├── Random: 500,000-1,000,000 IOPS
├── Latency: 20-100 microseconds
└── Compare SATA SSD: 550 MB/s, 100K IOPS

Device naming:
├── /dev/nvme0 - First NVMe controller
├── /dev/nvme0n1 - First namespace
├── /dev/nvme0n1p1 - First partition
└── Multiple namespaces supported
```

### Storage Area Networks (SAN)

```
SAN:

├── Network-attached block storage
├── Appears as local disk to OS
├── High performance (Fibre Channel, iSCSI)
├── Centralized management
├── Shared between servers
├── Snapshots and replication
└── Used in enterprise/data centers

Types:
├── Fibre Channel (FC)
│   ├── Dedicated network
│   ├── Very high performance
│   └── Expensive
├── iSCSI
│   ├── SCSI over TCP/IP
│   ├── Standard Ethernet
│   └── Cost-effective
└── FCoE
    ├── Fibre Channel over Ethernet
    ├── Converged network
    └── Complex
```

### Network-Attached Storage (NAS)

```
NAS:

├── File-level storage over network
├── Accessed via NFS, SMB/CIFS
├── Simpler than SAN
├── Shared file access
├── Common in home/small business
└── Examples: Synology, QNAP, FreeNAS

Protocols:
├── NFS (Unix/Linux)
├── SMB/CIFS (Windows)
├── AFP (legacy macOS)
└── SMB3 (modern, all platforms)
```

### Cloud Storage

```
Cloud Storage:

├── Object storage (S3, Blob, GCS)
├── Block storage (EBS, Persistent Disk)
├── File storage (EFS, Filestore)
├── Managed by provider
├── Pay for what you use
└── Different access patterns

Examples:
├── AWS: S3, EBS, EFS
├── Azure: Blob, Managed Disks, Files
├── GCP: Cloud Storage, Persistent Disk, Filestore
└── Integration with OS varies
```

---

## Summary

Disk organization spans from physical hardware through logical structures to OS abstractions. Understanding this hierarchy—physical disks, partitions, volumes, and file systems—is essential for understanding how data is stored and retrieved.

### Key Points

1. HDDs store data magnetically on rotating platters organized into tracks, sectors, and cylinders.
2. SSDs store data in NAND flash with no moving parts, requiring different management (wear leveling, garbage collection).
3. LBA (Logical Block Addressing) replaced CHS addressing, simplifying disk access.
4. Partitioning divides disks into independent regions, each potentially holding a different file system.
5. MBR and GPT are the two partition table formats—GPT is modern, supporting larger disks and more partitions.
6. LVM adds flexibility through physical volumes, volume groups, and logical volumes.
7. RAID provides redundancy and performance through various levels (0, 1, 5, 6, 10).
8. The OS presents disks as block devices with buffering, caching, and naming conventions.
9. Formatting creates file systems on partitions (high-level formatting) after partitioning.
10. Modern technologies (NVMe, SAN, NAS, cloud) extend traditional disk organization.

---

## Key Takeaways

1. Physical structure matters—HDD performance is dominated by seek time and rotational latency.
2. Sequential access is much faster than random on HDDs—allocation strategies leverage this.
3. SSDs change the rules—random access is as fast as sequential, but wear and erase blocks matter.
4. Partitions provide isolation—separate file systems for different purposes.
5. GPT is the modern standard—use it for new installations.
6. LVM adds flexibility—resize, snapshot, and manage storage dynamically.
7. RAID is about trade-offs—performance vs. redundancy vs. capacity.
8. The OS caches aggressively—page cache hides disk latency.
9. Block device naming can change—use UUIDs or labels in configuration files.
10. Storage technology evolves—NVMe, cloud, and new file systems continue to advance.

---

## Further Reading

- File-Allocation.md: How file systems allocate disk blocks
- File-System-Architecture.md: File system layers and VFS
- Inodes.md: Inode structure and metadata management
- Journaling.md: Consistency and crash recovery
- Introduction.md: Overview of file systems

