# File Allocation

## Introduction

File allocation is the set of methods and strategies that a file system uses to assign disk blocks to files and to track where each file's data resides on the storage device. When you save a file, the file system must decide which disk blocks will hold the data, record that decision in the file's metadata (typically the inode), and ensure the blocks can be located efficiently when the file is read later.

This is one of the most fundamental design decisions in any file system. The allocation method determines:

- How efficiently disk space is used (minimizing waste)
- How fast files can be accessed (sequential and random)
- How large files can grow (maximum file size)
- How well the system handles fragmentation (internal and external)
- How complex the file system implementation becomes

This document explores the three classic file allocation methods—contiguous, linked, and indexed—along with modern hybrid approaches used in real-world file systems like ext4, NTFS, and APFS.

---

## The Allocation Problem

### What Needs to Be Solved

When a file is created and written to, the file system must answer several questions:

```
Fundamental Allocation Questions:

1. Where should the file's data blocks be placed?
   ├── Which specific disk blocks?
   ├── Should they be contiguous or scattered?
   └── How to choose among free blocks?

2. How should the file's location be recorded?
   ├── In the file's metadata (inode)?
   ├── In a separate table?
   └── Distributed across blocks?

3. How do we find the data later?
   ├── Given a byte offset, which block?
   ├── How to traverse the block list?
   └── How to handle large files?

4. How do we manage free space?
   ├── Track available blocks
   ├── Allocate on file growth
   ├── Free on file deletion
   └── Minimize fragmentation
```

### Key Terminology

| Term                   | Definition                                            |
|------------------------|-------------------------------------------------------|
| Block                  | Smallest addressable unit of disk storage (typically 4 KB) |
| Logical Block          | File system's view of a block (may differ from physical) |
| Extent                 | A contiguous range of blocks allocated together         |
| Fragmentation          | Scattering of file data across non-contiguous blocks   |
| Internal Fragmentation  | Wasted space within an allocated block                 |
| External Fragmentation  | Free space scattered in small pieces                   |
| File Offset            | Byte position within a file                            |
| Block Pointer          | Reference to a specific disk block                     |

### Design Trade-offs

```
Allocation Method Trade-offs:

Space Efficiency:
├── Minimal waste of disk space
├── Efficient handling of small files
├── Efficient handling of large files
└── Support for file growth

Access Performance:
├── Fast sequential access
├── Fast random access
├── Minimal seek time (HDD)
├── Minimal read-modify-write (SSD)
└── Efficient read-ahead

Scalability:
├── Maximum file size supported
├── Maximum number of files
├── Maximum file system size
└── Efficient metadata overhead

Reliability:
├── Recovery from crashes
├── Detection of corruption
├── Redundancy of metadata
└── Consistent state maintenance
```

---

## Contiguous Allocation

### Concept

In contiguous allocation, each file occupies a set of consecutive blocks on the disk. The file's metadata records only the starting block and the length (number of blocks).

```
Contiguous Allocation:

File A: starts at block 5, length 4
├── Blocks 5, 6, 7, 8

File B: starts at block 12, length 3
├── Blocks 12, 13, 14

Disk Layout:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │ 8  │ 9  │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│free│free│free│free│free│ A  │ A  │ A  │ A  │free│
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 10 │ 11 │ 12 │ 13 │ 14 │ 15 │ 16 │ 17 │ 18 │ 19 │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│free│free│ B  │ B  │ B  │free│free│free│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘

Metadata:
File A: {start: 5, length: 4}
File B: {start: 12, length: 3}
```

### Allocation Strategy

```
Contiguous Allocation Strategies:

First Fit:
├── Find first gap large enough
├── Simple, fast
├── Fast allocation, may waste space at start
└── Example: Search from lowest address

Best Fit:
├── Find smallest gap that fits
├── Minimizes wasted space
├── Slower (must search entire list)
└── Leaves many small gaps

Worst Fit:
├── Find largest gap
├── Leaves largest remaining gap
├── Slower, generally poor performance
└── Rarely used

Next Fit:
├── Like first fit, but starts from last allocation
├── More even distribution
├── May miss better fits
└── Used in some systems
```

### Accessing Data

Contiguous allocation provides excellent access characteristics:

```
Accessing Byte at Offset O in File:

Block number = start_block + (O / block_size)
Offset within block = O % block_size

Example:
├── File A starts at block 5
├── Block size = 4 KB
├── Access byte 10,000

Block number = 5 + (10000 / 4096) = 5 + 2 = 7
Offset in block = 10000 % 4096 = 1808

Access:
├── Read block 7
├── Extract bytes starting at offset 1808
└── Fast and simple!
```

### Advantages

```
Contiguous Allocation Advantages:

1. Simple Metadata
   ├── Only need start block and length
   ├── Minimal metadata overhead
   ├── Very small inode
   └── Easy to implement

2. Excellent Sequential Access
   ├── Data is physically contiguous
   ├── Minimal disk seeks (HDD)
   ├── Prefetching is trivial
   ├── Best possible read performance
   └── Ideal for streaming

3. Excellent Random Access
   ├── Direct calculation of block location
   ├── No traversal needed
   ├── O(1) access to any block
   └── Fast direct access

4. Minimal Overhead
   ├── No pointer blocks needed
   ├── No fragmentation within file
   ├── No metadata blocks
   └── Efficient for large files
```

### Disadvantages

```
Contiguous Allocation Disadvantages:

1. External Fragmentation
   ├── Free space becomes fragmented over time
   ├── No single contiguous block large enough
   ├── Files cannot be allocated despite free space
   ├── Requires defragmentation
   └── Major problem for long-running systems

   Example:
   After creating and deleting many files:
   [Free 2][Used 5][Free 3][Used 2][Free 4][Used 3]...
   A 10-block file cannot be allocated!

2. File Growth Problem
   ├── File may need to grow beyond initial allocation
   ├── Adjacent blocks may be occupied
   ├── Options:
   │   ├── Copy entire file to larger space (expensive)
   │   ├── Allocate additional extent (creates fragmentation)
   │   └── Fail the operation
   └── Major limitation

3. Maximum File Size
   ├── Limited by largest contiguous free space
   ├── File size limited by disk size
   ├── User must estimate size in advance
   └── Cannot easily grow beyond estimate

4. Requires Advance Knowledge
   ├── User must specify size at creation
   ├── Overestimate → wasted space
   ├── Underestimate → file cannot grow
   └── Inconvenient for dynamic content
```

### When Contiguous is Used

```
Appropriate Uses for Contiguous Allocation:

1. Read-Only Media
   ├── CD-ROM, DVD
   ├── Write-once media
   ├── Data laid out optimally in advance
   └── No fragmentation possible

2. Boot Images
   ├── Kernel files
   ├── Initramfs
   ├── Contiguous improves boot speed
   └── Allocated during system install

3. Video/Audio Recording
   ├── Streaming data
   ├── Sequential access pattern
   ├── Contiguous prevents buffering issues
   └── Pre-allocation possible

4. Databases
   ├── Tablespaces allocated upfront
   ├── Contiguous for performance
   ├── Managed by database, not general FS
   └── Pre-allocated with known size

5. Extents in Modern File Systems
   ├── Contiguous runs within larger files
   ├── Best of both worlds
   ├── Extent-based allocation (ext4, XFS)
   └── Discussed later in this document
```

---

## Linked Allocation

### Concept

In linked allocation, each file is stored as a linked list of disk blocks. Each block contains both data and a pointer to the next block. The file's metadata points to the first block.

```
Linked Allocation:

File A: start at block 5
├── Block 5: [data...][next=8]
├── Block 8: [data...][next=12]
├── Block 12: [data...][next=3]
├── Block 3: [data...][next=null]

Disk Layout:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │ 8  │ 9  │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│free│free│free│ A  │free│ A  │free│free│ A  │free│
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
                         ↑              ↑
                    First block    Next block
                         │              │
                         └──next=8──────┘

┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 10 │ 11 │ 12 │ 13 │ 14 │ 15 │ 16 │ 17 │ 18 │ 19 │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│free│free│ A  │free│free│free│free│free│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
          ↑
    Last block (next=null)

Metadata:
File A: {start: 5}
```

### Accessing Data

Linked allocation requires traversing the chain:

```
Accessing Byte at Offset O in File A:

Block size = 4 KB
Number of blocks to skip = O / 4096

Traversal:
├── Start at block 5 (first block)
├── Read block 5, follow pointer to block 8
├── Read block 8, follow pointer to block 12
├── Read block 12, follow pointer to block 3
└── Read desired block, extract byte

For offset 10000: skip 2 blocks
├── Block 5 → Block 8 → Block 12
├── 3 block reads required (including metadata lookups)
└── Much slower than contiguous!

Sequential Access:
├── Read block 5, get data + pointer
├── Read block 8, get data + pointer
├── Read block 12, get data + pointer
└── Efficient if reads are batched
```

### The Pointer Overhead Problem

Every block must reserve space for the next pointer:

```
Pointer Overhead:

Block size: 4 KB = 4096 bytes
Pointer size: 4 bytes (32-bit block number)

Data per block: 4096 - 4 = 4092 bytes
Overhead: 4 / 4096 = 0.098%

For large files:
├── File size: 1 GB = 262,144 blocks
├── Overhead: 262,144 × 4 = 1 MB
├── Percentage: 0.098%
└── Acceptable for large files

For small files:
├── File size: 100 bytes
├── Still uses 1 full block: 4096 bytes
├── Overhead: 3996 bytes wasted = 97.6%
└── Terrible for small files!
```

### The FAT Approach

The File Allocation Table (FAT) file system improves on linked allocation by separating the pointers from the data:

```
FAT (File Allocation Table) Approach:

Separate table instead of in-block pointers:
├── FAT is an array indexed by block number
├── FAT[i] = next block in chain (or END, or FREE)
├── Data blocks contain only data
├── Table can be cached in memory
└── Faster traversal (no disk reads for pointers)

Example FAT:

Block:  0    1    2    3    4    5    6    7    8    9
FAT:   FREE FREE FREE 12   FREE 8    FREE FREE 12   FREE
                   ↓                    ↓
              (block 3)             (block 5)

Block: 10   11   12   13   14   ...
FAT:  FREE FREE END  FREE FREE ...

File A: starts at block 5
├── Block 5 → FAT[5]=8 → Block 8
├── Block 8 → FAT[8]=12 → Block 12
├── Block 12 → FAT[12]=END
└── Chain: 5 → 8 → 12

Key advantage:
├── FAT can be cached in RAM
├── Traversal is memory-only
├── Much faster than disk-based pointers
└── This is how FAT32, exFAT work

Cost:
├── FAT size proportional to disk size
├── Large disks → large FAT
├── For 1 TB disk with 4 KB blocks:
│   ├── 268 million blocks
│   ├── 4 bytes per entry
│   └── FAT size: ~1 GB
└── Must keep in memory or page
```

### Advantages

```
Linked Allocation Advantages:

1. No External Fragmentation
   ├── Any free block can be used
   ├── Files can grow dynamically
   ├── No need for contiguous space
   └── Better space utilization

2. Simple File Growth
   ├── Allocate any free block
   ├── Update pointer
   ├── No need to relocate
   └── Files can grow indefinitely

3. No Size Estimation
   ├── No need to know size in advance
   ├── File grows one block at a time
   ├── Natural for dynamic content
   └── User-friendly

4. Simple Metadata
   ├── Only need starting block
   ├── Small inode
   ├── Minimal metadata overhead
   └── Easy to implement
```

### Disadvantages

```
Linked Allocation Disadvantages:

1. Poor Random Access
   ├── Must traverse chain from start
   ├── O(n) access to nth block
   ├── Multiple disk reads (without FAT caching)
   ├── Terrible for random access patterns
   └── Cannot skip ahead efficiently

2. Pointer Overhead
   ├── Space wasted on pointers
   ├── Especially bad for small files
   ├── Reduces effective block size
   └── Wasted space accumulates

3. Reliability Issues
   ├── Corrupted pointer breaks entire chain
   ├── All data after corruption is lost
   ├── No redundancy
   └── Single point of failure

4. Poor Locality
   ├── Blocks scattered across disk
   ├── Many seeks for sequential access (HDD)
   ├── Slow for large files
   └── Bad for performance

5. FAT Size Problem
   ├── FAT for large disks is huge
   ├── Must be kept in memory
   ├── Competes with data cache
   └── Limits scalability
```

### When Linked is Used

```
Appropriate Uses for Linked Allocation:

1. FAT File Systems
   ├── FAT32, exFAT, FAT16
   ├── USB drives, SD cards
   ├── Simple embedded systems
   └── FAT cached in memory mitigates issues

2. Small Files
   ├── Files smaller than one block
   ├── Pointer overhead less significant
   ├── Chain is short
   └── Acceptable performance

3. Log-Structured File Systems
   ├── Append-only writes
   ├── Sequential access pattern
   ├── Linked structure natural
   └── LFS uses this concept

4. Distributed File Systems
   ├── Blocks may be on different servers
   ├── Linked approach extensible
   ├── Pointer to next location
   └── Example: some P2P systems

5. Recovery/Repair Tools
   ├── Rebuilding file chains
   ├── Salvaging data from corruption
   ├── Following chain structure
   └── Used in forensics
```

---

## Indexed Allocation

### Concept

In indexed allocation, each file has an index block (or multiple index blocks) that contains pointers to all the file's data blocks. The file's metadata points to the index block.

```
Indexed Allocation:

File A: index block at 20
├── Index block contains:
│   ├── Block 5
│   ├── Block 8
│   ├── Block 12
│   └── Block 3
├── Data blocks: 5, 8, 12, 3

Disk Layout:
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │ 8  │ 9  │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│free│free│free│ A  │free│ A  │free│free│ A  │free│
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
                  ↑              ↑              ↑
               data block     data block    data block

┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 10 │ 11 │ 12 │ 13 │ 14 │ 15 │ 16 │ 17 │ 18 │ 19 │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│free│free│ A  │free│free│free│free│free│free│free│
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
          ↑
       data block

┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ 20 │ 21 │ 22 │ 23 │ 24 │ 25 │ 26 │ 27 │ 28 │ 29 │
├────┼────┼────┼────┼────┼────┼────┼────┼────┼────┤
│ 5  │ 8  │ 12 │ 3  │ -- │ -- │ -- │ -- │ -- │ -- │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
  ↑
Index block for A

Metadata:
File A: {index_block: 20}
```

### Accessing Data

Indexed allocation provides fast random access:

```
Accessing Byte at Offset O in File A:

Block size = 4 KB
Block index = O / 4096

1. Read index block (block 20)
2. Look up pointer at position [block_index]
   ├── block_index=0 → pointer=5
   ├── block_index=1 → pointer=8
   ├── block_index=2 → pointer=12
   └── block_index=3 → pointer=3
3. Read data block
4. Extract byte at offset O % 4096

For offset 10000:
├── block_index = 2
├── Read index block (20) → get pointer 12
├── Read data block 12
├── Extract byte
└── 2 disk reads (constant time!)
```

### Handling Large Files

A single index block can only hold a limited number of pointers. For large files, multiple levels of indirection are used:

```
Single Index Block Limit:
├── Block size: 4 KB = 4096 bytes
├── Pointer size: 4 bytes
├── Pointers per block: 4096 / 4 = 1024
├── Max file size: 1024 × 4 KB = 4 MB
└── Too small for modern files!

Multi-Level Indexing (Unix-style):

Inode contains:
├── 12 direct pointers → 12 × 4 KB = 48 KB
├── 1 single indirect pointer → 1024 × 4 KB = 4 MB
├── 1 double indirect pointer → 1024² × 4 KB = 4 GB
└── 1 triple indirect pointer → 1024³ × 4 KB = 4 TB

Total max file size: ~4 TB

Structure:
├── Direct: Pointer directly to data block
├── Single Indirect: Pointer to block of pointers to data
├── Double Indirect: Pointer to block of pointers to blocks of pointers
└── Triple Indirect: Three levels of pointers

Access time:
├── Direct: 1 disk read (data)
├── Single Indirect: 2 disk reads (index + data)
├── Double Indirect: 3 disk reads
└── Triple Indirect: 4 disk reads
```

### Visualizing Multi-Level Indexing

```
Unix-Style Multi-Level Indexed Allocation:

Inode:
┌─────────────────┐
│ Direct[0]  ─────┼───▶ Data Block 0
│ Direct[1]  ─────┼───▶ Data Block 1
│ ...             │
│ Direct[11] ─────┼───▶ Data Block 11
├─────────────────┤
│ Single Ind. ────┼───▶ Index Block ──┬──▶ Data Block
│                 │                   ├──▶ Data Block
│                 │                   ├──▶ ...
│                 │                   └──▶ Data Block
├─────────────────┤
│ Double Ind. ────┼───▶ Index Block ──┬──▶ Index Block ──┬──▶ Data
│                 │                   │                 ├──▶ Data
│                 │                   │                 └──▶ ...
│                 │                   └──▶ Index Block ──┬──▶ Data
│                 │                                     └──▶ ...
├─────────────────┤
│ Triple Ind. ────┼───▶ Index Block ──▶ Index Block ──▶ Index Block ──▶ Data
└─────────────────┘
```

### Advantages

```
Indexed Allocation Advantages:

1. Fast Random Access
   ├── O(1) access to any block (mostly)
   ├── Direct pointer lookup
   ├── No chain traversal
   └── Excellent for databases

2. No External Fragmentation
   ├── Blocks can be scattered
   ├── Any free block can be used
   ├── Files grow without relocation
   └── Good space utilization

3. Supports Large Files
   ├── Multi-level indexing scales
   ├── Max file size in TB range
   ├── Extensible to any size
   └── Handles modern file sizes

4. No Size Estimation Required
   ├── File grows dynamically
   ├── Index blocks allocated as needed
   ├── Natural file growth
   └── User-friendly

5. Efficient for Sparse Files
   ├── Unallocated blocks have null pointers
   ├── File "size" can exceed actual blocks
   ├── Efficient storage for sparse files
   └── Holes represented as null pointers
```

### Disadvantages

```
Indexed Allocation Disadvantages:

1. Index Block Overhead
   ├── Extra block per file (minimum)
   ├── Wasted for tiny files
   ├── A 1-byte file uses 1 data + 1 index block
   ├── Significant for many small files
   └── Better than linked, worse than contiguous

2. Index Block Access Cost
   ├── Must read index block first
   ├── Extra disk I/O per access
   ├── Caching mitigates this
   ├── Still slower than contiguous
   └── Double read for each access

3. Complexity
   ├── More complex than linked
   ├── Multi-level indexing is intricate
   ├── More code to implement
   └── More bugs to find

4. Multi-Level Overhead
   ├── Large files need many index blocks
   ├── Multiple indirection levels
   ├── More metadata blocks
   └── Deeper access chains

5. Index Block Allocation
   ├── Must allocate index block on file creation
   ├── Even for empty file
   ├── Wastes one block
   └── Special handling for small files
```

### When Indexed is Used

```
Appropriate Uses for Indexed Allocation:

1. Unix File Systems
   ├── Traditional Unix (System V, BSD)
   ├── Multi-level indexed inodes
   ├── Well-understood design
   └── Foundation for ext family

2. Database Systems
   ├── Random access patterns
   ├── Large files
   ├── Need fast seek
   └── Index structure natural

3. Modern File Systems
   ├── ext4 uses extents (not pure indexing)
   ├── But inode structure is indexed-based
   ├── Evolved from indexed allocation
   └── Hybrid approaches

4. Network File Systems
   ├── Random access over network
   ├── Index structure allows parallelism
   ├── Efficient remote access
   └── Example: NFS

5. Sparse Files
   ├── Efficient representation of holes
   ├── Null pointers in index block
   ├── Used for VMs, disk images
   └── Natural fit
```

---

## Comparison of Allocation Methods

### Comprehensive Comparison Table

| Aspect                     | Contiguous | Linked   | Indexed   |
|---------------------------|------------|----------|-----------|
| Metadata Size             | Very small | Very small | Medium-Large |
| Sequential Access          | Excellent  | Moderate | Good      |
| Random Access              | Excellent  | Poor     | Excellent  |
| File Growth                | Difficult  | Easy     | Easy      |
| External Frag. Problem     | None       | None     | None      |
| Internal Frag.             | Yes        | Yes      | Yes       |
| Size Estimation Required    | Not required | Not required | Not required |
| Max File Size              | Limited    | Large    | Very Large |
| Reliability                | Good       | Poor     | Good      |
| Complexity                 | Simple     | Simple   | Complex   |
| Small File Overhead        | Low        | Low      | Medium    |
| Large File Performance      | Excellent  | Poor     | Good      |

### Visual Comparison

```
Allocation Method Comparison:

File Layout: 
Contiguous:  [A A A A A A A A][B B B B][C C C C C C]
             Perfectly ordered, contiguous runs

Linked:      [A]→[A]→[A]→[A]→[A]  scattered
             [B]→[B]→[B]→[B]      with pointers
             [C]→[C]→[C]→[C]→[C]→[C]

Indexed:     [Inode A]→[A index]→[A A A A]
             [Inode B]→[B index]→[B B B]
             [Inode C]→[C index]→[C C C C C]

Access Time Comparison (for block N):

Contiguous:  O(1) - direct calculation
Linked:      O(N) - traverse chain
Indexed:     O(1) to O(log N) - depends on level

Growth Capability:
Contiguous:  Hard - needs new contiguous space
Linked:      Easy - add block anywhere
Indexed:     Easy - add pointer + block
```

### When to Use Which

```
Decision Guide:

Use Contiguous When:
├── File size known in advance
├── Sequential access dominates
├── Read-only media (CD/DVD)
├── Performance is critical
└── Example: boot images, video files

Use Linked When:
├── File grows unpredictably
├── Sequential access dominates
├── Simple implementation required
├── Small files
└── Example: FAT32 USB drives

Use Indexed When:
├── Random access needed
├── Files can be large
├── Modern general-purpose system
├── Balance of performance and features
└── Example: Unix/Linux file systems
```

---

## Modern Approaches: Extents

Modern file systems (ext4, XFS, APFS, NTFS) use extents—a hybrid approach that combines benefits of contiguous and indexed allocation.

### The Extent Concept

An extent is a contiguous range of blocks described by a single (start, length) pair:

```
Extent Concept:

Traditional indexed allocation:
├── Block 5
├── Block 6
├── Block 7
├── Block 8
└── 4 separate pointers for 4 contiguous blocks

Extent-based allocation:
└── Extent {start: 5, length: 4}
    Single descriptor for 4 contiguous blocks

Advantages:
├── Much less metadata for contiguous files
├── Extent describes many blocks with one entry
├── Better performance (contiguous runs)
├── Still handles fragmentation (multiple extents)
└── Best of both worlds
```

### Extent Tree

Large files have multiple extents organized in a tree:

```
Extent Tree Structure (ext4):

Inode:
├── Extent header
├── Extent 1: {start: 100, length: 50}   ← 50 blocks at 100
├── Extent 2: {start: 500, length: 200}  ← 200 blocks at 500
├── Extent 3: {start: 1000, length: 30}  ← 30 blocks at 1000
└── ...

For files with many extents:
├── Root extent node (in inode)
├── Points to extent index nodes
├── Eventually points to leaf extents
└── B-tree structure for scalability

Example (B-tree):
                Root (in inode)
                /      |      \
        Index       Index      Index
        /   \       /   \      /   \
      Ext   Ext   Ext   Ext  Ext   Ext
```

### Extent Advantages

```
Extent-Based Allocation Advantages:

1. Reduced Metadata
   ├── One descriptor for many blocks
   ├── File with 1000 contiguous blocks needs 1 entry
   ├── Massive reduction in metadata overhead
   └── Scales to very large files

2. Better Performance
   ├── Contiguous runs enable read-ahead
   ├── Fewer seeks (HDD)
   ├── Better erase-block alignment (SSD)
   └── Efficient large I/O

3. Handles Fragmentation
   ├── Multiple extents allowed per file
   ├── Files can grow and fragment
   ├── Still more efficient than block-by-block
   └── Graceful degradation

4. Natural for Large Files
   ├── Databases, media files
   ├── Few extents dominate
   ├── Near-contiguous performance
   └── Minimal metadata

5. Extent Tree Scalability
   ├── B-tree allows many extents
   ├── Log-time lookup
   ├── Efficient for huge files
   └── ext4 supports up to 4 billion extents
```

### Real-World Examples

```
Extents in Modern File Systems:

ext4:
├── Extent-based by default (since Linux 2.6.23)
├── Inode contains extent tree root
├── 4 extents in inode directly
├── Additional extents in tree
├── Max extent: 32768 blocks (128 MB with 4KB blocks)
├── Max file: 16 TB
└── 4 extents per leaf node

XFS:
├── Extent-based (from the start)
├── B+tree for extents
├── More extents than ext4
├── Better for very large files
├── Used in high-performance systems
└── Max file: 8 EB (theoretical)

NTFS:
├── Extent-based (called "runs")
├── MFT entry contains first runs
├── Additional runs in external MFT
├── Supports sparse files (compressed runs)
└── Used in Windows

APFS:
├── Extent-based
├── Copy-on-write
├── B-tree structure
├── Optimized for SSD
└── Used in macOS/iOS

Btrfs:
├── Extent-based
├── Copy-on-write
├── B-tree for all metadata
├── Checksums on data and metadata
└── Advanced features (snapshots, RAID)
```

### Extent vs. Traditional Indexed

```
Comparison:

Traditional Indexed (Unix FFS):
├── 12 direct pointers per inode
├── Single indirect (1024 pointers)
├── Double indirect (1024² pointers)
├── Triple indirect (1024³ pointers)
├── Max file: ~4 TB (with 4KB blocks)
├── Metadata per file: Large for many-block files
└── Example: 1 MB file needs 12 direct + 256 indirect pointers

Extent-Based (ext4):
├── Extent tree in inode
├── 4 extents directly in inode
├── Additional extents in tree nodes
├── Max file: 16 TB (ext4)
├── Metadata per file: Small (1 extent = 128 MB)
└── Example: 1 MB file needs 1 extent

For 1 GB file:
├── Traditional: 12 direct + 1024 indirect + 256 × double indirect
│   └── Thousands of pointers = many metadata blocks
├── Extent: ~8 extents (128 MB each)
│   └── Fits in inode directly!
└── Extent is dramatically more efficient
```

---

## Implementation Details

### Data Structures

```c
// Allocation method implementations

// 1. CONTIGUOUS ALLOCATION
struct contiguous_file {
    uint32_t start_block;    // First block
    uint32_t length;         // Number of blocks
    // File data at blocks start_block ... start_block+length-1
};

// 2. LINKED ALLOCATION (basic)
struct linked_block {
    uint8_t data[BLOCK_SIZE - sizeof(uint32_t)];
    uint32_t next_block;     // 0 = end of chain
};

struct linked_file {
    uint32_t first_block;
};

// 3. LINKED ALLOCATION (FAT)
struct fat_file {
    uint32_t first_block;
    // FAT array indexed by block number
    // FAT[block] = next block (or END, FREE)
};

// 4. INDEXED ALLOCATION (Unix-style)
#define DIRECT_PTRS 12

struct indexed_inode {
    uint32_t direct[DIRECT_PTRS];      // Direct block pointers
    uint32_t single_indirect;          // → block of 1024 pointers
    uint32_t double_indirect;          // → block of pointers
    uint32_t triple_indirect;          // → three-level indirection
    uint32_t file_size;
    // ... other metadata
};

// 5. EXTENT-BASED ALLOCATION
struct extent {
    uint32_t start_block;    // First block of extent
    uint32_t length;         // Number of blocks
    // Total blocks described: length
};

#define EXTENTS_PER_INODE 4

struct extent_inode {
    uint16_t magic;          // Extent tree magic
    uint16_t entries;        // Number of extents
    uint16_t max;            // Max entries
    uint16_t depth;          // Tree depth
    struct extent extents[EXTENTS_PER_INODE];
    // If depth > 0, these are index nodes pointing to child nodes
};
```

### Contiguous Allocation Implementation

```c
// Simple contiguous allocation

struct free_extent {
    uint32_t start;
    uint32_t length;
    struct free_extent *next;
};

// Allocate contiguous blocks
int allocate_contiguous(struct free_extent **free_list, uint32_t size, 
                        uint32_t *start_out) {
    struct free_extent *prev = NULL;
    struct free_extent *curr = *free_list;
    
    while (curr != NULL) {
        if (curr->length >= size) {
            // Found a suitable extent
            *start_out = curr->start;
            
            if (curr->length == size) {
                // Remove entire extent
                if (prev == NULL) {
                    *free_list = curr->next;
                } else {
                    prev->next = curr->next;
                }
                free(curr);
            } else {
                // Split the extent
                curr->start += size;
                curr->length -= size;
            }
            return 0;  // Success
        }
        prev = curr;
        curr = curr->next;
    }
    
    return -1;  // No suitable extent found
}

// Free contiguous blocks
void free_contiguous(struct free_extent **free_list, 
                     uint32_t start, uint32_t length) {
    struct free_extent *new_extent = malloc(sizeof(struct free_extent));
    new_extent->start = start;
    new_extent->length = length;
    
    // Insert in sorted order (by start)
    struct free_extent *prev = NULL;
    struct free_extent *curr = *free_list;
    
    while (curr != NULL && curr->start < start) {
        prev = curr;
        curr = curr->next;
    }
    
    new_extent->next = curr;
    if (prev == NULL) {
        *free_list = new_extent;
    } else {
        prev->next = new_extent;
    }
    
    // Coalesce with next
    if (curr != NULL && start + length == curr->start) {
        new_extent->length += curr->length;
        new_extent->next = curr->next;
        free(curr);
    }
    
    // Coalesce with previous
    if (prev != NULL && prev->start + prev->length == start) {
        prev->length += new_extent->length;
        prev->next = new_extent->next;
        free(new_extent);
    }
}
```

### Indexed Allocation Implementation

```c
// Unix-style indexed allocation

#define BLOCK_SIZE 4096
#define PTRS_PER_BLOCK (BLOCK_SIZE / sizeof(uint32_t))  // 1024
#define DIRECT_PTRS 12
#define INDIRECT_PTRS 1

// Calculate maximum file size
uint64_t max_file_size() {
    uint64_t direct = DIRECT_PTRS * BLOCK_SIZE;
    uint64_t single = PTRS_PER_BLOCK * BLOCK_SIZE;
    uint64_t double = (uint64_t)PTRS_PER_BLOCK * PTRS_PER_BLOCK * BLOCK_SIZE;
    uint64_t triple = (uint64_t)PTRS_PER_BLOCK * PTRS_PER_BLOCK * 
                      PTRS_PER_BLOCK * BLOCK_SIZE;
    return direct + single + double + triple;
    // = 48 KB + 4 MB + 4 GB + 4 TB ≈ 4 TB
}

// Get block pointer for file offset
uint32_t get_block(struct indexed_inode *inode, uint64_t offset, int allocate) {
    uint64_t block_num = offset / BLOCK_SIZE;
    
    if (block_num < DIRECT_PTRS) {
        // Direct pointer
        if (inode->direct[block_num] == 0 && allocate) {
            inode->direct[block_num] = allocate_block();
        }
        return inode->direct[block_num];
    }
    block_num -= DIRECT_PTRS;
    
    if (block_num < PTRS_PER_BLOCK) {
        // Single indirect
        if (inode->single_indirect == 0) {
            if (!allocate) return 0;
            inode->single_indirect = allocate_block();
            // Zero the block
        }
        
        uint32_t *indirect = read_block(inode->single_indirect);
        if (indirect[block_num] == 0 && allocate) {
            indirect[block_num] = allocate_block();
            write_block(inode->single_indirect, indirect);
        }
        return indirect[block_num];
    }
    block_num -= PTRS_PER_BLOCK;
    
    if (block_num < (uint64_t)PTRS_PER_BLOCK * PTRS_PER_BLOCK) {
        // Double indirect
        uint32_t l1_index = block_num / PTRS_PER_BLOCK;
        uint32_t l2_index = block_num % PTRS_PER_BLOCK;
        
        if (inode->double_indirect == 0) {
            if (!allocate) return 0;
            inode->double_indirect = allocate_block();
        }
        
        uint32_t *l1 = read_block(inode->double_indirect);
        if (l1[l1_index] == 0) {
            if (!allocate) return 0;
            l1[l1_index] = allocate_block();
            write_block(inode->double_indirect, l1);
        }
        
        uint32_t *l2 = read_block(l1[l1_index]);
        if (l2[l2_index] == 0 && allocate) {
            l2[l2_index] = allocate_block();
            write_block(l1[l1_index], l2);
        }
        return l2[l2_index];
    }
    block_num -= (uint64_t)PTRS_PER_BLOCK * PTRS_PER_BLOCK;
    
    // Triple indirect (similar structure)
    // ... implementation omitted for brevity
    
    return 0;  // Beyond max file size
}
```

### Extent-Based Implementation

```c
// Extent-based allocation (simplified ext4-like)

#define EXTENTS_PER_INODE 4
#define MAX_EXTENT_LENGTH 32768

struct extent {
    uint32_t start_block;    // First block of extent
    uint32_t length;         // Number of blocks
    // Total blocks described: length
};

struct extent_header {
    uint16_t magic;         // 0xF30A
    uint16_t entries;       // Number of valid entries
    uint16_t max;           // Max entries
    uint16_t depth;         // Tree depth (0 = leaf)
    uint32_t generation;
};

struct extent_index {
    uint32_t block;         // Block containing child node
    uint32_t leaf_lo;       // Low 32 bits of logical block
    uint16_t leaf_hi;       // High 16 bits of logical block
    uint16_t unused;
};

// Find extent containing logical block
struct extent *find_extent(struct extent_header *header, 
                           uint64_t logical_block) {
    if (header->depth == 0) {
        // Leaf node - search extents
        struct extent *extents = (struct extent *)(header + 1);
        for (int i = 0; i < header->entries; i++) {
            if (logical_block >= extents[i].start_block &&
                logical_block < extents[i].start_block + extents[i].length) {
                return &extents[i];
            }
        }
        return NULL;
    } else {
        // Index node - find child
        struct extent_index *index = (struct extent_index *)(header + 1);
        for (int i = 0; i < header->entries; i++) {
            uint64_t child_start = ((uint64_t)index[i].leaf_hi << 32) | 
                                    index[i].leaf_lo;
            uint64_t next_start;
            
            if (i + 1 < header->entries) {
                next_start = ((uint64_t)index[i+1].leaf_hi << 32) | 
                             index[i+1].leaf_lo;
            } else {
                next_start = UINT64_MAX;
            }
            
            if (logical_block >= child_start && logical_block < next_start) {
                // Recurse into child
                struct extent_header *child = read_block(index[i].block);
                return find_extent(child, logical_block);
            }
        }
        return NULL;
    }
}
```

---

## Free Space Management

Allocation methods interact with free space management. Common approaches:

### 1. Bitmap

```
Bitmap Approach:

One bit per block:
├── 0 = free
├── 1 = allocated

Example (16 blocks):
Bitmap: 0 1 1 0 0 1 1 1 0 0 1 0 0 0 1 0
Block:  0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15

Advantages:
├── Simple
├── Easy to find free blocks
├── Space-efficient (1 bit per block)
└── For 1 TB disk with 4KB blocks: 32 MB bitmap

Disadvantages:
├── Must scan to find contiguous runs
├── Inefficient for finding large extents
└── Best for small block allocation
```

### 2. Free List

```
Free List Approach:

Linked list of free blocks:
├── Each free block contains pointer to next
├── Head points to first free block
├── Simple but poor locality
└── Rarely used for main allocation

Free extent list:
├── List of (start, length) pairs
├── More efficient for extents
├── Sorted for coalescing
└── Used in extent-based file systems
```

### 3. Space Maps (Modern)

```
Space Map Approach (ZFS, Btrfs):

Sophisticated data structures:
├── B-tree of allocated extents
├── B-tree of free extents
├── Metaslab allocator
├── Reduces fragmentation
└── Efficient for large disks

Benefits:
├── Fast allocation
├── Reduced fragmentation
├── Scales to huge disks
├── Handles snapshots
└── Used in modern file systems
```

---

## Fragmentation

### Internal Fragmentation

Wasted space within allocated blocks:

```
Internal Fragmentation:

File size: 100 bytes
Block size: 4096 bytes
Allocated: 1 block (4096 bytes)
Wasted: 4096 - 100 = 3996 bytes (97.6%)

For many small files:
├── 1000 files of 100 bytes each
├── 1000 blocks allocated (4 MB)
├── Actual data: 100 KB
├── Wasted: 3.9 MB
└── Internal fragmentation is significant!

Mitigations:
├── Smaller blocks (more overhead)
├── Block suballocation (tail packing)
├── Extent merging for small files
└── Inline data in inode
```

### External Fragmentation

Free space scattered in small pieces:

```
External Fragmentation:

Initial state: [Free 100 blocks]
After allocations and deallocations:
[Used 5][Free 3][Used 10][Free 2][Used 8][Free 5]...

Now allocating a 15-block file fails, even though
there are 10 free blocks total (3+2+5).

Extent-based file systems:
├── Can use multiple extents
├── File becomes fragmented but works
├── Performance degrades
└── Defragmentation can help

Contiguous allocation:
├── Cannot handle this case
├── Requires defragmentation
├── Or use larger allocation units
└── Major limitation
```

### Defragmentation

```
Defragmentation:

Purpose:
├── Rearrange files for contiguity
├── Improve sequential access
├── Consolidate free space
└── Restore performance

Types:
├── File defragmentation
│   ├── Move file blocks together
│   ├── Update pointers
│   └── Example: Windows defrag
│
├── Free space defragmentation
│   ├── Move files to consolidate free space
│   ├── Create large contiguous free areas
│   └── Example: ext4 e4defrag
│
└── Online defragmentation
    ├── Defrag while system running
    ├── Copy-on-write friendly
    └── Example: Btrfs balance, XFS online defrag

Modern approach:
├── Extents reduce fragmentation impact
├── SSDs don't benefit from defrag
├── Some file systems avoid fragmentation by design
└── Less critical than in the past
```

---

## Real-World File System Examples

### ext4 Allocation

```
ext4 Allocation:

Method: Extent-based
├── Extent tree in inode
├── 4 extents inline in inode
├── Additional extents in tree nodes
├── Max extent: 32768 blocks (128 MB)
└── Max file: 16 TB

Features:
├── Delayed allocation (allocate on write-back)
├── Multi-block allocation (allocate runs)
├── Extent merging
├── Online defragmentation
└── Flexible block groups

Allocation strategy:
├── Orlov allocator for directories
├── Block groups for locality
├── Preallocation for streaming
└── Fallocate for explicit allocation
```

### NTFS Allocation

```
NTFS Allocation:

Method: Extent-based (called "runs")
├── MFT entry for each file
├── First runs in MFT entry
├── Additional runs in external MFT
├── Sparse runs for holes
└── Max file: 16 TB (practical)

Features:
├── Cluster-based (default 4 KB)
├── Sparse files
├── Compressed files
├── Alternate data streams
└── Journaling

Allocation strategy:
├── Best-fit for small files
├── First-fit for large files
├── Bitmap for free clusters
└── Reserved MFT zone
```

### XFS Allocation

```
XFS Allocation:

Method: Extent-based with B+tree
├── Extent list for each file
├── B+tree for large files
├── Extent size: variable, up to 1 GB
├── Max file: 8 EB (theoretical)
└── Optimized for large files

Features:
├── Allocation groups (parallelism)
├── Delayed allocation
├── Preallocation
├── Online defragmentation
├── Realtime subvolume
└── Journaling

Allocation strategy:
├── B+tree for free space (by size and by block)
├── Striped allocation for RAID
├── Alignment to stripe boundaries
└── Efficient for large files and directories
```

### FAT32 Allocation

```
FAT32 Allocation:

Method: Linked (via FAT)
├── FAT table in memory
├── Each entry = next block in chain
├── No extent support
├── Max file: 4 GB
└── Max volume: 2 TB (practical)

Features:
├── Simple design
├── Wide compatibility
├── No journaling
├── No permissions
└── No metadata beyond basic

Allocation strategy:
├── First-fit from FAT
├── FAT cached in memory
├── Cluster-based (typically 4 KB)
└── Wastes space for small files
```

---

## Common Misconceptions

### Misconception 1: "Contiguous allocation is always best"

**Reality:** Contiguous allocation has excellent performance but severe problems with fragmentation and file growth. Modern file systems use extent-based approaches that provide contiguity when possible while handling fragmentation gracefully.

### Misconception 2: "Linked allocation is obsolete"

**Reality:** While pure linked allocation has poor random access performance, the concept is used in FAT file systems (which cache the FAT in memory) and log-structured file systems. It's not obsolete—it’s evolved.

### Misconception 3: "Indexed allocation requires more disk space"

**Reality:** Indexed allocation has index block overhead, but this is often offset by efficient handling of large files and reduced fragmentation. For large files, indexed allocation can use less metadata than alternative approaches.

### Misconception 4: "Defragmentation is always beneficial"

**Reality:** Defragmentation benefits HDDs by reducing seeks. On SSDs, defragmentation provides no benefit (no seek time) and can reduce drive life by causing extra writes. Modern SSDs should not be defragmented.

### Misconception 5: "Extents are just large blocks"

**Reality:** Extents are variable-length descriptions of contiguous block runs. They're more flexible than fixed-size blocks and more efficient than per-block pointers. An extent of length 1000 uses one descriptor instead of 1000 pointers.

---

## Hands-On Exercises

### Exercise 1: Allocation Calculation

**Problem:** A disk has 1 TB capacity with 4 KB blocks. Calculate:

- Number of blocks
- Size of FAT for linked allocation
- Size of bitmap for free space
- Number of pointers needed for a 1 GB file (indexed)

**Solution:**

```
1. Number of blocks:
   1 TB = 1,099,511,627,776 bytes
   1,099,511,627,776 / 4096 = 268,435,456 blocks

2. FAT size:
   268,435,456 blocks × 4 bytes/entry = 1,073,741,824 bytes
   = 1 GB (must be cached in memory)

3. Bitmap size:
   268,435,456 bits / 8 = 33,554,432 bytes = 32 MB

4. Pointers for 1 GB file:
   1 GB / 4 KB = 262,144 blocks
   
   With direct pointers (12):
   262,144 - 12 = 262,132 blocks remaining
   
   With single indirect (1024):
   262,132 / 1024 ≈ 256 indirect blocks
   
   Metadata blocks: 1 single indirect index block
                    (plus inode)
   Pointers: 12 + 262,132 = 262,144 pointers
   
   With extents (128 MB max):
   1 GB / 128 MB = 8 extents needed
   8 extent descriptors (small!)
```

### Exercise 2: Fragmentation Analysis

**Problem:** A disk has the following free extents: [5 blocks, 3 blocks, 8 blocks, 2 blocks, 12 blocks]. A new file needs 15 blocks.

- Can contiguous allocation satisfy this?
- Can extent-based allocation satisfy this?

**Solution:**

```
Contiguous allocation:
├── Need 15 contiguous blocks
├── Largest free extent: 12 blocks
├── Cannot satisfy request
└── Result: Allocation fails

Extent-based allocation:
├── Can use multiple extents
├── Allocate: 12 + 3 = 15 blocks (2 extents)
├── Or: 8 + 5 + 2 = 15 blocks (3 extents)
└── Result: Allocation succeeds (with fragmentation)

This demonstrates the key advantage of extent-based
allocation over pure contiguous allocation.
```

### Exercise 3: Access Time Calculation

**Problem:** Given the Unix inode structure (12 direct, 1 single, 1 double, 1 triple indirect), calculate the access time for byte 5,000,000 in a file. Assume 4 KB blocks and 4-byte pointers.

**Solution:**

```
Block size: 4096 bytes
Byte offset: 5,000,000
Block number: 5,000,000 / 4096 = 1220

Direct blocks: 0-11 (12 blocks)
Block 1220 > 11, so not direct.

Remaining: 1220 - 12 = 1208
Single indirect: 1024 blocks (0-1023 in indirect)
Block 1208 > 1023, so not single indirect.

Remaining: 1208 - 1024 = 184
Double indirect: 1024 × 1024 = 1,048,576 blocks
Block 184 is within double indirect range.

Double indirect access:
├── Level 1 index: 184 / 1024 = 0
├── Level 2 index: 184 % 1024 = 184
├── Read inode (cached typically)
├── Read double indirect block (level 1)
├── Read single indirect block (level 2)
└── Read data block

Total disk reads (cold cache): 4
├── inode: 1 (usually cached)
├── level 1 index: 1
├── level 2 index: 1
└── data block: 1

Warm cache: 1 read (just the data block)
```

---

## Summary

File allocation determines how disk blocks are assigned to files and how those assignments are

