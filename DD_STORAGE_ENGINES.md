# Storage Engines — Deep Dive

> How databases physically store, retrieve, and maintain data on disk.

**Prerequisites:** [Data Management Overview](./DATA_MANAGEMENT.md), [Data Storage & Access](./03_DATA_STORAGE_AND_ACCESS.md)
**Related:** [Sharding & Partitioning](./DD_SHARDING_PARTITIONING.md), [Scaling & Infrastructure](./09_SCALING_AND_INFRASTRUCTURE.md)
**Estimated study time:** 3-4 hours

---

## Table of Contents

1. [Context & Problem Statement](#1-context--problem-statement)
2. [Storage Fundamentals](#2-storage-fundamentals)
3. [B-Tree Family](#3-b-tree-family)
4. [LSM-Tree Family](#4-lsm-tree-family)
5. [Write-Ahead Log (WAL)](#5-write-ahead-log-wal)
6. [Bloom Filters](#6-bloom-filters)
7. [Amplification Factors](#7-amplification-factors)
8. [Production Tuning](#8-production-tuning)
9. [Hybrid Approaches](#9-hybrid-approaches)
10. [Interview Articulation](#10-interview-articulation)
11. [Decision Framework](#11-decision-framework)
12. [Quick Reference Card](#12-quick-reference-card)
13. [References](#references)

---

## 1. Context & Problem Statement

### The Fundamental Challenge

Databases must balance two conflicting access patterns:

```mermaid
flowchart LR
    subgraph "The Tension"
        DISK["Disk Characteristics"]
        SEQ["Sequential I/O: ~500 MB/s"]
        RAND["Random I/O: ~200 IOPS<br/>(~0.8 MB/s with 4KB pages)"]

        DISK --> SEQ
        DISK --> RAND
    end

    subgraph "Workload Needs"
        WRITE["Writes want sequential<br/>(fast append)"]
        READ["Reads want sorted<br/>(fast lookup)"]
    end

    SEQ -.->|"favors"| WRITE
    RAND -.->|"requires"| READ
```

**The core trade-off**: Optimizing for writes (sequential appends) makes reads expensive (scan everything). Optimizing for reads (sorted structure) makes writes expensive (maintain order).

### Why Storage Engines Matter

| Decision | Impact |
|----------|--------|
| Wrong engine for workload | 10-100x performance difference |
| Misconfigured buffer pool | Constant disk thrashing |
| Ignoring write amplification | Premature SSD wear, high latency |
| No understanding of compaction | Unexpected latency spikes |

### The Two Dominant Paradigms

| Paradigm | Optimizes For | Examples |
|----------|---------------|----------|
| **B-Tree** | Reads, point lookups | PostgreSQL, MySQL InnoDB, Oracle |
| **LSM-Tree** | Writes, sequential workloads | Cassandra, RocksDB, LevelDB, HBase |

---

## 2. Storage Fundamentals

### Disk I/O Characteristics

Understanding storage media is essential for engine design:

| Medium | Sequential Read | Random Read (4KB) | Latency |
|--------|-----------------|-------------------|---------|
| HDD | 150-200 MB/s | 100-200 IOPS | 5-10 ms |
| SATA SSD | 500-600 MB/s | 50K-100K IOPS | 0.1 ms |
| NVMe SSD | 3-7 GB/s | 500K-1M IOPS | 0.02 ms |
| Memory | 50+ GB/s | N/A | 0.0001 ms |

**Key Insight**: Even with SSDs, sequential access is significantly faster than random. Storage engines that convert random writes to sequential writes gain substantial performance.

### Page-Based Storage

Databases read and write data in fixed-size **pages** (typically 4KB-16KB):

```mermaid
flowchart TB
    subgraph "Page Structure (8KB example)"
        HEADER["Page Header (24 bytes)<br/>page_id, checksum, LSN"]
        SLOTS["Slot Array<br/>(grows down)"]
        FREE["Free Space"]
        DATA["Row Data<br/>(grows up)"]

        HEADER --> SLOTS
        SLOTS --> FREE
        FREE --> DATA
    end
```

**Why pages?**
- Matches OS and disk block sizes
- Enables efficient caching
- Allows atomic writes (with checksums)
- Simplifies buffer management

### The Buffer Pool

All databases use an in-memory cache called the **buffer pool**:

```mermaid
flowchart LR
    subgraph "Database Process"
        APP["Query Executor"]
        BP["Buffer Pool<br/>(Cached Pages)"]
    end

    subgraph "Storage"
        DISK["Disk Pages"]
    end

    APP -->|"1. Request page"| BP
    BP -->|"2a. Cache hit"| APP
    BP -->|"2b. Cache miss"| DISK
    DISK -->|"3. Load page"| BP
    BP -->|"4. Return data"| APP

    BP -->|"Background flush"| DISK
```

**Buffer Pool Management**:
- **LRU/Clock**: Evict least recently used pages
- **Dirty page tracking**: Know which pages need writing
- **Pin counting**: Prevent eviction of in-use pages
- **Prefetching**: Read ahead for sequential scans

---

## 3. B-Tree Family

### B-Tree vs B+Tree

Most databases use **B+Trees**, not plain B-Trees:

```mermaid
flowchart TB
    subgraph "B-Tree (Original)"
        BT_ROOT["Root<br/>Keys + Data Pointers"]
        BT_INT["Internal<br/>Keys + Data Pointers"]
        BT_LEAF["Leaf<br/>Keys + Data Pointers"]

        BT_ROOT --> BT_INT
        BT_INT --> BT_LEAF
    end

    subgraph "B+Tree (Database Standard)"
        BP_ROOT["Root<br/>Keys only (routing)"]
        BP_INT["Internal<br/>Keys only (routing)"]
        BP_LEAF["Leaves<br/>Keys + All Data"]
        BP_LINK["Leaf Links (→)"]

        BP_ROOT --> BP_INT
        BP_INT --> BP_LEAF
        BP_LEAF --- BP_LINK
    end
```

| Aspect | B-Tree | B+Tree |
|--------|--------|--------|
| Data location | All nodes | Leaves only |
| Internal nodes | Keys + data | Keys only (more keys per node) |
| Leaf linking | No | Yes (enables range scans) |
| Range queries | Tree traversal | Sequential leaf scan |
| Cache efficiency | Lower | Higher (internal nodes smaller) |

### B+Tree Structure Deep Dive

```mermaid
flowchart TB
    subgraph "B+Tree with Branching Factor 4"
        ROOT["[30 | 60 | 90]"]

        N1["[10 | 20]"]
        N2["[40 | 50]"]
        N3["[70 | 80]"]
        N4["[100 | 110]"]

        L1["5,8,10"]
        L2["15,18,20"]
        L3["25,28,30"]
        L4["35,38,40"]
        L5["45,48,50"]
        L6["55,58,60"]
        L7["65,68,70"]
        L8["75,78,80"]
        L9["85,88,90"]
        L10["95,98,100"]
        L11["105,108,110"]
        L12["115,118,120"]

        ROOT --> N1
        ROOT --> N2
        ROOT --> N3
        ROOT --> N4

        N1 --> L1
        N1 --> L2
        N1 --> L3
        N2 --> L4
        N2 --> L5
        N2 --> L6
        N3 --> L7
        N3 --> L8
        N3 --> L9
        N4 --> L10
        N4 --> L11
        N4 --> L12

        L1 -.-> L2 -.-> L3 -.-> L4 -.-> L5 -.-> L6
    end
```

**Key Properties**:
- **Branching factor (B)**: Number of children per node (typically 100-500)
- **Height**: O(log_B N) — very shallow even for billions of rows
- **Balanced**: All leaves at same depth
- **Sorted**: Keys in order within and across nodes

**Height Analysis**:

| Rows | B=100 | B=500 |
|------|-------|-------|
| 10K | 2 | 2 |
| 1M | 3 | 2 |
| 100M | 4 | 3 |
| 10B | 5 | 4 |

With B=500 and 10 billion rows, only 4 page reads needed for any lookup!

### Page Splits and Merges

**Page Split** (on insert when page is full):

```mermaid
flowchart LR
    subgraph "Before Split"
        FULL["[10,20,30,40,50]<br/>FULL - inserting 35"]
    end

    subgraph "After Split"
        LEFT["[10,20,30]"]
        RIGHT["[35,40,50]"]
        PARENT["Parent gets key 35"]
    end

    FULL --> LEFT
    FULL --> RIGHT
    RIGHT -.-> PARENT
```

```python
def split_page(page, new_key, new_value):
    """
    Split a full B+Tree leaf page.

    1. Create new page
    2. Move half the entries to new page
    3. Insert new key in appropriate page
    4. Add pointer to new page in parent
    5. If parent full, recursively split
    """
    mid = len(page.entries) // 2
    new_page = Page()

    # Move upper half to new page
    new_page.entries = page.entries[mid:]
    page.entries = page.entries[:mid]

    # Insert new entry in correct page
    if new_key < new_page.entries[0].key:
        page.insert(new_key, new_value)
    else:
        new_page.insert(new_key, new_value)

    # Promote middle key to parent
    separator_key = new_page.entries[0].key
    parent.insert_child(separator_key, new_page)

    return page, new_page
```

**Page Merge** (on delete when page too empty):

```mermaid
flowchart LR
    subgraph "Before Merge"
        SPARSE1["[10]<br/>< 50% full"]
        SPARSE2["[20,30]<br/>Sibling"]
    end

    subgraph "After Merge"
        MERGED["[10,20,30]<br/>Combined"]
    end

    SPARSE1 --> MERGED
    SPARSE2 --> MERGED
```

### B+Tree Write Path

```mermaid
sequenceDiagram
    participant App as Application
    participant BP as Buffer Pool
    participant WAL as Write-Ahead Log
    participant Disk as Data Files

    App->>BP: INSERT (key, value)
    BP->>BP: Find target leaf page
    BP->>WAL: Log the change
    WAL->>WAL: fsync (durable)
    BP->>BP: Modify page in memory
    BP-->>App: Ack (committed)

    Note over BP,Disk: Background checkpoint
    BP->>Disk: Flush dirty pages
    Note over WAL: Truncate old WAL
```

**Write Amplification**: A single logical write may cause:
1. WAL write (sequential)
2. Page modification
3. Page split → parent modification → cascading splits
4. Background flush of dirty pages

Typical: 2-10x amplification

### B+Tree Read Path

```mermaid
sequenceDiagram
    participant App as Application
    participant BP as Buffer Pool
    participant Disk as Data Files

    App->>BP: SELECT WHERE key = X
    BP->>BP: Check root page (cached)
    BP->>BP: Navigate to internal page

    alt Page in buffer pool
        BP-->>App: Return from cache
    else Page not cached
        BP->>Disk: Read page
        Disk-->>BP: Page data
        BP->>BP: Cache page (evict LRU if needed)
        BP-->>App: Return data
    end
```

### Complexity Analysis

| Operation | Time | I/O (worst) | Notes |
|-----------|------|-------------|-------|
| Point lookup | O(log_B N) | O(log_B N) | Tree traversal |
| Range scan | O(log_B N + K) | O(log_B N + K/F) | Find start + scan K rows, F rows per page |
| Insert | O(log_B N) | O(log_B N) | May cause splits |
| Update | O(log_B N) | O(log_B N) | Find + modify in place |
| Delete | O(log_B N) | O(log_B N) | May cause merges |

*B = branching factor, N = total rows, K = result size, F = fill factor*

---

## 4. LSM-Tree Family

### Core Concept

Log-Structured Merge Tree converts random writes to sequential writes by:
1. Buffering writes in memory (MemTable)
2. Flushing to immutable sorted files (SSTables)
3. Background merging (compaction) to maintain read performance

```mermaid
flowchart TB
    subgraph "LSM-Tree Architecture"
        subgraph "Memory"
            MT["MemTable<br/>(Sorted: Red-Black Tree / Skip List)"]
            IMT["Immutable MemTable<br/>(Being flushed)"]
        end

        subgraph "Disk Level 0"
            SST0A["SSTable 0-A"]
            SST0B["SSTable 0-B"]
            SST0C["SSTable 0-C"]
        end

        subgraph "Disk Level 1"
            SST1A["SSTable 1-A"]
            SST1B["SSTable 1-B"]
        end

        subgraph "Disk Level 2"
            SST2["SSTable 2<br/>(Largest level)"]
        end
    end

    WRITE[Write] --> MT
    MT -->|"Full"| IMT
    IMT -->|"Flush"| SST0A
    SST0A -->|"Compaction"| SST1A
    SST1A -->|"Compaction"| SST2
```

### MemTable

In-memory sorted structure for recent writes:

```python
class MemTable:
    """
    In-memory write buffer using a skip list or red-black tree.

    Properties:
    - Sorted by key for efficient range scans
    - O(log N) insert, lookup, delete
    - Typically sized 64MB - 256MB
    - Flushed to SSTable when full
    """

    def __init__(self, max_size_bytes=64 * 1024 * 1024):
        self.data = SortedDict()  # Or skip list
        self.size_bytes = 0
        self.max_size = max_size_bytes

    def put(self, key: bytes, value: bytes) -> bool:
        """Insert or update key-value pair."""
        old_size = self._entry_size(key, self.data.get(key))
        new_size = self._entry_size(key, value)

        self.data[key] = value
        self.size_bytes += (new_size - old_size)

        return self.size_bytes >= self.max_size  # Returns True if flush needed

    def get(self, key: bytes) -> Optional[bytes]:
        """Lookup key. Returns None if not found."""
        return self.data.get(key)

    def delete(self, key: bytes):
        """Delete by inserting tombstone marker."""
        self.put(key, TOMBSTONE)

    def flush_to_sstable(self, path: str) -> SSTable:
        """Write sorted contents to immutable SSTable file."""
        with SSTableWriter(path) as writer:
            for key, value in self.data.items():
                writer.write(key, value)
            writer.finish()  # Write index, bloom filter, metadata
        return SSTable(path)
```

### SSTable (Sorted String Table)

Immutable, sorted file format:

```
┌─────────────────────────────────────────────────────────────────┐
│                        SSTable File Format                       │
├─────────────────────────────────────────────────────────────────┤
│  Data Blocks                                                     │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐      │
│  │ Block 0     │ Block 1     │ Block 2     │ ...         │      │
│  │ (sorted KVs)│ (sorted KVs)│ (sorted KVs)│             │      │
│  └─────────────┴─────────────┴─────────────┴─────────────┘      │
├─────────────────────────────────────────────────────────────────┤
│  Index Block                                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ [first_key_block0 → offset0]                            │    │
│  │ [first_key_block1 → offset1]                            │    │
│  │ [first_key_block2 → offset2]                            │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  Bloom Filter (for fast negative lookups)                        │
├─────────────────────────────────────────────────────────────────┤
│  Footer (index offset, bloom offset, checksum, version)          │
└─────────────────────────────────────────────────────────────────┘
```

### LSM-Tree Write Path

```mermaid
sequenceDiagram
    participant App as Application
    participant WAL as Write-Ahead Log
    participant MT as MemTable
    participant L0 as Level 0
    participant L1 as Level 1

    App->>WAL: Append entry (sequential)
    App->>MT: Insert into MemTable
    MT-->>App: Ack (fast!)

    Note over MT: MemTable fills up (64MB)

    MT->>L0: Flush as SSTable
    Note over L0: New SSTable added<br/>(may overlap with others)

    Note over L0,L1: Background compaction
    L0->>L1: Merge overlapping SSTables
    Note over L1: Sorted, non-overlapping
```

**Key Insight**: Writes only touch memory and sequential WAL. No random I/O!

### LSM-Tree Read Path

```mermaid
flowchart TD
    READ["Read Key 'X'"] --> MT{MemTable?}
    MT -->|"Found"| RETURN[Return Value]
    MT -->|"Not Found"| IMT{Immutable<br/>MemTable?}

    IMT -->|"Found"| RETURN
    IMT -->|"Not Found"| L0{Level 0<br/>SSTables?}

    L0 -->|"Found"| RETURN
    L0 -->|"Not Found"| L1{Level 1?}

    L1 -->|"Found"| RETURN
    L1 -->|"Not Found"| L2{Level 2?}

    L2 -->|"Found"| RETURN
    L2 -->|"Not Found"| DEEPER[Continue deeper...<br/>or NOT FOUND]

    subgraph "Per SSTable"
        BLOOM["1. Check Bloom filter"]
        INDEX["2. Binary search index"]
        BLOCK["3. Read data block"]

        BLOOM --> INDEX --> BLOCK
    end
```

**Read Amplification**: Must check multiple levels. Bloom filters help skip SSTables that definitely don't contain the key.

### Compaction Strategies

#### Size-Tiered Compaction (STCS)

Merge SSTables of similar size:

```mermaid
flowchart LR
    subgraph "Size-Tiered Compaction"
        S1["SST<br/>10MB"]
        S2["SST<br/>12MB"]
        S3["SST<br/>11MB"]
        S4["SST<br/>9MB"]

        MERGE["Merge when<br/>4 similar-sized"]

        S1 --> MERGE
        S2 --> MERGE
        S3 --> MERGE
        S4 --> MERGE

        MERGE --> BIG["SST<br/>~40MB"]
    end
```

| Aspect | Size-Tiered |
|--------|-------------|
| Write amplification | Lower (~4-10x) |
| Space amplification | Higher (up to 2x) |
| Read amplification | Higher (many overlapping files) |
| Compaction I/O | Bursty |
| Best for | Write-heavy workloads |

#### Leveled Compaction (LCS)

Fixed-size levels with strict sorting:

```mermaid
flowchart TB
    subgraph "Leveled Compaction"
        L0["Level 0: May overlap<br/>4 SSTables max"]
        L1["Level 1: 10MB total<br/>Non-overlapping"]
        L2["Level 2: 100MB total<br/>Non-overlapping (10x L1)"]
        L3["Level 3: 1GB total<br/>Non-overlapping (10x L2)"]

        L0 -->|"Compact"| L1
        L1 -->|"Compact"| L2
        L2 -->|"Compact"| L3
    end
```

| Aspect | Leveled |
|--------|---------|
| Write amplification | Higher (~10-30x) |
| Space amplification | Lower (~10-20%) |
| Read amplification | Lower (1 file per level) |
| Compaction I/O | Steady |
| Best for | Read-heavy, space-constrained |

#### Comparison

| Factor | Size-Tiered | Leveled |
|--------|-------------|---------|
| Write amp | 4-10x | 10-30x |
| Read amp | High | Low |
| Space amp | ~2x | ~1.1x |
| Compaction pattern | Bursty | Continuous |
| Latency variance | Higher | Lower |

### Complexity Analysis

| Operation | Time | I/O (typical) | Notes |
|-----------|------|---------------|-------|
| Write | O(log N) | O(1) amortized | MemTable + sequential WAL |
| Point lookup | O(L × log N) | O(L) | Check L levels |
| Range scan | O(L × log N + K) | O(L + K/F) | Merge L iterators |
| Compaction | O(N log N) | O(N) | Background, amortized |

*L = number of levels, N = total entries, K = result size, F = entries per block*

---

## 5. Write-Ahead Log (WAL)

### Purpose

The WAL ensures durability: committed transactions survive crashes.

```mermaid
flowchart LR
    subgraph "Without WAL"
        W1["Write to memory"] --> C1["Crash!"] --> L1["Data lost"]
    end

    subgraph "With WAL"
        W2["Write to WAL (fsync)"] --> W3["Write to memory"] --> C2["Crash!"]
        C2 --> R2["Replay WAL"] --> RECOVER["Data recovered"]
    end
```

### WAL Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                        WAL File Format                           │
├─────────────────────────────────────────────────────────────────┤
│  Record 1: [LSN=1] [Type=INSERT] [Table=users] [Data=...]       │
│  Record 2: [LSN=2] [Type=UPDATE] [Table=orders] [Data=...]      │
│  Record 3: [LSN=3] [Type=COMMIT] [TxnID=42]                     │
│  Record 4: [LSN=4] [Type=INSERT] [Table=items] [Data=...]       │
│  ...                                                             │
└─────────────────────────────────────────────────────────────────┘

LSN = Log Sequence Number (monotonically increasing)
```

### WAL Write Protocol

```python
def write_with_wal(key, value, wal, memtable):
    """
    Durable write using WAL.

    Protocol:
    1. Append to WAL
    2. fsync WAL (ensures durability)
    3. Apply to in-memory structure
    4. Acknowledge to client

    On crash recovery:
    - Replay WAL from last checkpoint
    - Rebuild in-memory state
    """
    # Step 1: Write to WAL
    lsn = wal.append(WriteRecord(key, value))

    # Step 2: Ensure durability
    wal.fsync()  # Critical! Data is now durable

    # Step 3: Apply to memory
    memtable.put(key, value)

    # Step 4: Return success
    return lsn
```

### Checkpoint Strategy

Periodically flush dirty pages to reduce recovery time:

```mermaid
sequenceDiagram
    participant WAL as Write-Ahead Log
    participant BP as Buffer Pool
    participant Data as Data Files

    Note over WAL: Continuous writes accumulate

    Note over BP: Checkpoint triggered<br/>(time or size threshold)

    BP->>BP: Identify dirty pages
    BP->>Data: Flush dirty pages
    Data-->>BP: Flush complete

    BP->>WAL: Record checkpoint LSN
    WAL->>WAL: Truncate old entries

    Note over WAL: Only need to replay<br/>from checkpoint on recovery
```

**Checkpoint Tradeoffs**:
- More frequent: Faster recovery, more I/O overhead
- Less frequent: Slower recovery, less I/O overhead

---

## 6. Bloom Filters

### Purpose

Bloom filters provide fast negative lookups: "Definitely not present" vs "Maybe present."

```mermaid
flowchart LR
    QUERY["Query: key=X"] --> BLOOM{"Bloom Filter"}

    BLOOM -->|"Definitely NO"| SKIP["Skip SSTable<br/>(no I/O!)"]
    BLOOM -->|"Maybe YES"| CHECK["Check SSTable<br/>(may be false positive)"]
```

### How It Works

```mermaid
flowchart TB
    subgraph "Bloom Filter Structure"
        BIT["Bit Array: [0,0,0,0,0,0,0,0,0,0]"]
    end

    subgraph "Insert 'apple'"
        H1["hash1('apple') % 10 = 2"]
        H2["hash2('apple') % 10 = 5"]
        H3["hash3('apple') % 10 = 8"]
        RESULT1["[0,0,1,0,0,1,0,0,1,0]"]
    end

    subgraph "Query 'banana'"
        Q1["hash1('banana') % 10 = 3"]
        Q2["hash2('banana') % 10 = 5"]
        Q3["hash3('banana') % 10 = 7"]
        CHECK1["Position 3 = 0 → NOT PRESENT"]
    end
```

### Implementation

```python
import mmh3  # MurmurHash

class BloomFilter:
    """
    Space-efficient probabilistic data structure.

    Properties:
    - No false negatives: if filter says "no", definitely not present
    - False positives possible: if filter says "yes", might not be present
    - Cannot delete entries (without counting variant)
    """

    def __init__(self, expected_items: int, fp_rate: float = 0.01):
        """
        Initialize bloom filter.

        Args:
            expected_items: Expected number of items
            fp_rate: Desired false positive rate (default 1%)
        """
        # Optimal size: m = -n * ln(p) / (ln(2)^2)
        self.size = int(-expected_items * math.log(fp_rate) / (math.log(2) ** 2))
        # Optimal hash count: k = (m/n) * ln(2)
        self.num_hashes = int((self.size / expected_items) * math.log(2))
        self.bit_array = [False] * self.size

    def add(self, item: str) -> None:
        """Add item to filter."""
        for seed in range(self.num_hashes):
            index = mmh3.hash(item, seed) % self.size
            self.bit_array[index] = True

    def might_contain(self, item: str) -> bool:
        """
        Check if item might be in set.

        Returns:
            False: Definitely not present
            True: Possibly present (check actual data)
        """
        for seed in range(self.num_hashes):
            index = mmh3.hash(item, seed) % self.size
            if not self.bit_array[index]:
                return False  # Definitely not present
        return True  # Maybe present
```

### Sizing Guidelines

| Items | FP Rate | Bits per Item | Total Size |
|-------|---------|---------------|------------|
| 1M | 1% | 9.6 | 1.2 MB |
| 1M | 0.1% | 14.4 | 1.8 MB |
| 10M | 1% | 9.6 | 12 MB |
| 100M | 1% | 9.6 | 120 MB |

**Formula**: Bits = -n × ln(p) / (ln(2))² ≈ 10 bits per item for 1% FP

### Impact on LSM Reads

Without Bloom filters:
- Check every SSTable: O(L × log N) I/O

With Bloom filters:
- Check filter first: O(L) memory operations
- Only read SSTables that might contain key
- Typical: 99% of unnecessary reads avoided

---

## 7. Amplification Factors

Three types of amplification measure storage engine efficiency:

### Write Amplification

Ratio of bytes written to storage vs bytes written by application.

```mermaid
flowchart LR
    subgraph "B-Tree Write Amplification"
        BT_APP["App writes 100 bytes"]
        BT_WAL["WAL: 100 bytes"]
        BT_PAGE["Page write: 8KB<br/>(even for 100 byte change)"]
        BT_TOTAL["Total: ~80x"]

        BT_APP --> BT_WAL --> BT_PAGE --> BT_TOTAL
    end

    subgraph "LSM Write Amplification"
        LSM_APP["App writes 100 bytes"]
        LSM_WAL["WAL: 100 bytes"]
        LSM_L0["Flush to L0: 100 bytes"]
        LSM_L1["Compact L0→L1: 100 bytes"]
        LSM_L2["Compact L1→L2: 100 bytes"]
        LSM_TOTAL["Total: 10-30x"]

        LSM_APP --> LSM_WAL --> LSM_L0 --> LSM_L1 --> LSM_L2 --> LSM_TOTAL
    end
```

| Engine | Write Amplification | Notes |
|--------|---------------------|-------|
| B-Tree | 2-10x | WAL + page rewrites |
| LSM (Size-Tiered) | 4-10x | Each level rewrite |
| LSM (Leveled) | 10-30x | More compaction |

**Why it matters**: High write amplification → faster SSD wear, higher latency

### Read Amplification

Number of reads required to satisfy one application read.

| Engine | Read Amplification | Notes |
|--------|---------------------|-------|
| B-Tree | 1-4 | Tree depth |
| LSM | 1-10+ | Check multiple levels |

### Space Amplification

Ratio of storage used vs logical data size.

| Engine | Space Amplification | Notes |
|--------|---------------------|-------|
| B-Tree | 1.5-2x | Page fragmentation, fill factor |
| LSM (Size-Tiered) | 2x | Old versions during compaction |
| LSM (Leveled) | 1.1-1.2x | Better space efficiency |

### The RUM Conjecture

**R**ead, **U**pdate (write), **M**emory (space) — optimize two, sacrifice one.

```mermaid
graph TD
    R["Read Optimal"]
    U["Update/Write Optimal"]
    M["Memory/Space Optimal"]

    R ---|"B-Tree"| M
    U ---|"LSM (Size-Tiered)"| R
    M ---|"LSM (Leveled)"| U

    CENTER["Can't optimize<br/>all three"]

    R --> CENTER
    U --> CENTER
    M --> CENTER
```

---

## 8. Production Tuning

### PostgreSQL (B-Tree)

```sql
-- Key configuration parameters
-- postgresql.conf

# Buffer pool (typically 25% of RAM)
shared_buffers = 4GB

# WAL settings
wal_level = replica
wal_buffers = 64MB
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
```

**Tuning Guidelines**:
- `shared_buffers`: Start at 25% RAM, increase for read-heavy
- `effective_cache_size`: Set to 75% RAM (query planner hint)
- `checkpoint_completion_target`: 0.9 spreads checkpoint I/O

### RocksDB (LSM)

```cpp
// Key configuration options
Options options;

// MemTable size (triggers flush)
options.write_buffer_size = 64 * 1024 * 1024;  // 64MB

// Number of MemTables before stalling
options.max_write_buffer_number = 3;

// Level 0 triggers
options.level0_file_num_compaction_trigger = 4;
options.level0_slowdown_writes_trigger = 20;
options.level0_stop_writes_trigger = 36;

// Compaction style
options.compaction_style = kCompactionStyleLevel;  // or kCompactionStyleUniversal

// Bloom filter
BlockBasedTableOptions table_options;
table_options.filter_policy.reset(NewBloomFilterPolicy(10));  // 10 bits per key

// Compression
options.compression = kSnappyCompression;
```

**Tuning Guidelines**:
- `write_buffer_size`: Larger = fewer flushes, more memory
- `level0_file_num_compaction_trigger`: Balance write stalls vs read amplification
- Bloom filter bits: 10 = 1% FP, 14 = 0.1% FP

### Cassandra (LSM)

```yaml
# cassandra.yaml

# MemTable settings
memtable_heap_space_in_mb: 2048
memtable_offheap_space_in_mb: 2048
memtable_flush_writers: 4

# Compaction
compaction_throughput_mb_per_sec: 64
concurrent_compactors: 4

# Bloom filter (per table)
# ALTER TABLE users WITH bloom_filter_fp_chance = 0.01;
```

---

## 9. Hybrid Approaches

### WiscKey: Separating Keys from Values

Store keys in LSM, values in separate log:

```mermaid
flowchart LR
    subgraph "Traditional LSM"
        TRAD["Key + Value together<br/>in LSM levels"]
    end

    subgraph "WiscKey"
        LSM["LSM: Keys + Value Pointers"]
        VLOG["Value Log: Actual values"]

        LSM --> VLOG
    end
```

**Benefits**: Smaller LSM → less compaction, lower write amplification
**Tradeoff**: Range scans require random value reads

### TiKV: Raft + RocksDB

```mermaid
flowchart TB
    subgraph "TiKV Architecture"
        RAFT["Raft Consensus"]
        ROCKS["RocksDB Storage"]

        RAFT -->|"Replicated writes"| ROCKS
    end
```

### CockroachDB: Pebble Engine

Custom LSM implementation with:
- Optimized for Raft log replay
- Range deletion tombstones
- Parallel compaction

---

## 10. Interview Articulation

### 30-Second Version

> "Storage engines are the core of how databases physically store data. B-Trees, used by PostgreSQL and MySQL, optimize for reads—they maintain sorted data with O(log N) lookups and support efficient range scans, but writes cause random I/O. LSM-Trees, used by Cassandra and RocksDB, optimize for writes—they buffer in memory and flush to sorted files, converting random writes to sequential. The tradeoff is write amplification from compaction and potentially slower reads that must check multiple levels. Choose B-Tree for read-heavy OLTP, LSM for write-heavy or time-series workloads."

### 2-Minute Version

> "Storage engines determine how databases physically organize data on disk. The two dominant approaches are B-Trees and LSM-Trees, each optimized for different access patterns.
>
> B-Trees maintain data in a sorted, balanced tree structure. They're read-optimized—a lookup requires traversing just 3-4 levels even for billions of rows, making point queries very fast. Range scans are efficient because leaves are linked. The downside is writes: updating data requires finding the correct leaf page and modifying it in place, which is random I/O. Page splits add overhead. PostgreSQL, MySQL InnoDB, and Oracle all use B-Trees.
>
> LSM-Trees take the opposite approach. All writes go to an in-memory buffer called the MemTable. When it fills up, it's flushed as an immutable sorted file called an SSTable. Background compaction merges these files to maintain read performance. The key insight is that writes are always sequential—either to the MemTable or appending SSTables. This makes LSM-Trees much faster for write-heavy workloads. Cassandra, RocksDB, and LevelDB use LSM-Trees.
>
> The tradeoffs involve three amplification factors. Write amplification: LSM has higher because data is rewritten during compaction (10-30x for leveled compaction). Read amplification: LSM has higher because reads may check multiple levels. Space amplification: depends on compaction strategy.
>
> Both use Write-Ahead Logs for durability—write to WAL, fsync, then apply to memory. Both use Bloom filters to optimize lookups—LSM-Trees especially benefit since they avoid checking SSTables that definitely don't contain the key.
>
> For tuning, key parameters are buffer pool size for B-Trees and MemTable size plus compaction settings for LSM-Trees."

### Common Follow-Up Questions

| Question | Key Points |
|----------|------------|
| "Why not just use LSM for everything?" | Read amplification, compaction pauses, complexity |
| "How does compaction affect latency?" | Can cause spikes; leveled is smoother than size-tiered |
| "What's write amplification?" | Ratio of physical writes to logical writes; affects SSD lifetime |
| "How do Bloom filters help?" | Avoid reading SSTables that don't contain the key |
| "What's the buffer pool?" | In-memory cache of disk pages; critical for B-Tree performance |
| "B-Tree vs B+Tree?" | B+Tree stores data only in leaves; better for range scans and cache efficiency |

---

## 11. Decision Framework

```mermaid
flowchart TD
    START[Choose Storage Engine] --> Q1{Workload Type?}

    Q1 -->|"Read-heavy<br/>(>10:1 read:write)"| BTREE[B-Tree<br/>PostgreSQL, MySQL]
    Q1 -->|"Write-heavy<br/>(>1:10 read:write)"| LSM[LSM-Tree<br/>Cassandra, RocksDB]
    Q1 -->|"Balanced"| Q2{Latency variance<br/>tolerance?}

    Q2 -->|"Low tolerance"| BTREE
    Q2 -->|"Can tolerate spikes"| LSM

    LSM --> Q3{Space or write<br/>efficiency priority?}

    Q3 -->|"Space efficiency"| LEVELED[Leveled Compaction]
    Q3 -->|"Write efficiency"| SIZETIERED[Size-Tiered Compaction]

    BTREE --> TUNE_BT["Tune: buffer pool,<br/>checkpoint interval"]
    LEVELED --> TUNE_LSM["Tune: L0 triggers,<br/>compaction throughput"]
    SIZETIERED --> TUNE_LSM
```

---

## 12. Quick Reference Card

### Engine Comparison

| Aspect | B-Tree | LSM-Tree |
|--------|--------|----------|
| Read performance | Excellent | Good (with Bloom filters) |
| Write performance | Good | Excellent |
| Write amplification | 2-10x | 10-30x (leveled) |
| Read amplification | 1-4x | 1-10x |
| Space amplification | 1.5-2x | 1.1-2x |
| Latency variance | Low | Higher (compaction) |
| Best for | OLTP, random reads | Write-heavy, time-series |

### Key Parameters

| Engine | Parameter | Typical Value | Impact |
|--------|-----------|---------------|--------|
| B-Tree | Buffer pool | 25% RAM | More = better read cache |
| B-Tree | Checkpoint interval | 5-15 min | More = faster crash, more I/O |
| LSM | MemTable size | 64-256 MB | Larger = fewer flushes |
| LSM | L0 compaction trigger | 4-8 files | Lower = more compaction, better reads |
| LSM | Bloom filter bits | 10 | 10 = 1% FP rate |

### Complexity Summary

| Operation | B-Tree | LSM-Tree |
|-----------|--------|----------|
| Point read | O(log N) | O(L log N) |
| Range scan | O(log N + K) | O(L log N + K) |
| Write | O(log N) | O(1) amortized |
| Space | O(N) | O(N) |

*N = entries, K = result size, L = levels*

### Common Pitfalls

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| Undersized buffer pool | High disk I/O, slow reads | Increase to 25%+ RAM |
| No Bloom filters | Slow LSM reads | Enable with 10 bits/key |
| Wrong compaction strategy | Write stalls or space bloat | Match to workload |
| Ignoring write amp | SSD wear, high latency | Monitor, tune compaction |

---

## References

### Academic Papers

- **Bayer & McCreight, 1970** — "Organization and Maintenance of Large Ordered Indices" — Original B-tree paper
- **O'Neil et al., 1996** — "The Log-Structured Merge-Tree (LSM-Tree)" — Original LSM-tree paper
- **Bloom, 1970** — "Space/Time Trade-offs in Hash Coding with Allowable Errors" — Bloom filter
- **Lu et al., 2016** — "WiscKey: Separating Keys from Values in SSD-conscious Storage" — Key-value separation
- **Athanassoulis et al., 2016** — "Designing Access Methods: The RUM Conjecture" — Amplification tradeoffs

### Production Documentation

- **PostgreSQL** — [Database Physical Storage](https://www.postgresql.org/docs/current/storage.html)
- **RocksDB** — [Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide)
- **Cassandra** — [Compaction](https://cassandra.apache.org/doc/latest/cassandra/operating/compaction/)
- **LevelDB** — [Implementation Notes](https://github.com/google/leveldb/blob/main/doc/impl.md)

---

## Revision History

| Date | Change |
|------|--------|
| 2025-01 | Initial deep-dive document with B-Tree, LSM-Tree mechanics, WAL durability, Bloom filters, amplification analysis |

---

## Navigation

**Parent:** [Data Management Overview](./DATA_MANAGEMENT.md)
**Related:** [Data Storage & Access](./03_DATA_STORAGE_AND_ACCESS.md), [Sharding & Partitioning](./DD_SHARDING_PARTITIONING.md)
**Previous:** [Consistent Hashing](./DD_CONSISTENT_HASHING.md)
**Index:** [README](./README.md)
