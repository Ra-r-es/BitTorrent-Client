# BitTorrent Client Implementation Guide
## Complete Step-by-Step Development Plan

---

## Phase 1: Foundation & Torrent File Parsing

### Task 1: Project Setup & Environment

**Objective:** Set up your Python project structure and dependencies

**Visual Structure:**
```
torrent_client/
│
├── 📄 torrent_parser.py      → Parse .torrent files
├── 📡 tracker_client.py      → Communicate with tracker
├── 🔌 peer_connection.py     → Connect to peers
├── 🧩 piece_manager.py       → Manage file pieces
├── 📥 download_manager.py    → Orchestrate downloads
└── ▶️  main.py               → Entry point

🐍 venv/                      → Virtual environment
📦 requirements.txt           → Dependencies
```

**Steps:**
- Create a virtual environment
- Install required libraries: `pip install requests`
- Create project structure

**Deliverable:** Working Python environment with folder structure

---

### Task 2: Implement Bencode Decoder

**Objective:** Parse the bencode format used in .torrent files

**Visual: Bencode Format Examples**
```
┌─────────────────────────────────────────┐
│ BENCODE FORMAT                          │
├─────────────────────────────────────────┤
│                                         │
│ Integer:   i42e           → 42          │
│            i-5e           → -5          │
│                                         │
│ String:    4:spam         → "spam"      │
│            10:hello world → "hello..."  │
│                                         │
│ List:      li1ei2ee       → [1, 2]      │
│            l4:spam3:egge  → ["spam","egg"]│
│                                         │
│ Dict:      d3:cow3:moo4:spam4:eggse    │
│            → {"cow":"moo", "spam":"eggs"}│
│                                         │
└─────────────────────────────────────────┘

Parsing Flow:
    Start
      ↓
   Read first character
      ↓
   ┌──┴──┐
   ↓     ↓     ↓     ↓
  'i'   'l'   'd'  digit
   ↓     ↓     ↓     ↓
  INT  LIST  DICT STRING
   ↓     ↓     ↓     ↓
   └─────┴─────┴─────┘
          ↓
    Return Python Object
```

**What to implement:**
- Function to decode integers (format: `i<number>e`)
- Function to decode strings (format: `<length>:<string>`)
- Function to decode lists (format: `l<items>e`)
- Function to decode dictionaries (format: `d<key><value>e`)

**Key concepts:**
- Bencode is the encoding used in torrent files
- Recursive parsing for nested structures

**Test:** Parse a sample torrent file and print its contents

**Deliverable:** `bencode_decode()` function that converts bencode to Python objects

---

### Task 3: Torrent File Parser

**Objective:** Extract metadata from .torrent files

**Visual: Torrent File Structure**
```
┌─────────────────────────────────────────────────────────┐
│ .TORRENT FILE STRUCTURE                                 │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  announce: "http://tracker.example.com:8080/announce"  │
│                                                         │
│  info:                                                  │
│    ├─ name: "ubuntu-20.04.iso"                         │
│    ├─ piece length: 262144  (256 KB)                   │
│    ├─ pieces: [hash1][hash2][hash3]...[hashN]          │
│    │          └─20 bytes each (SHA-1)                  │
│    └─ length: 2147483648  (2 GB)                       │
│                                                         │
└─────────────────────────────────────────────────────────┘

Info Hash Calculation:
    ┌──────────┐
    │   info   │  dictionary
    └────┬─────┘
         ↓
    [ bencode ]  encode to bytes
         ↓
    [ SHA-1   ]  hash function
         ↓
    [20 bytes ]  info_hash (torrent ID)
         ↓
    Used to identify torrent uniquely


File Pieces Visualization:
┌────────┬────────┬────────┬────────┬────┐
│Piece 0 │Piece 1 │Piece 2 │Piece 3 │... │
│256 KB  │256 KB  │256 KB  │256 KB  │    │
│hash₀   │hash₁   │hash₂   │hash₃   │    │
└────────┴────────┴────────┴────────┴────┘
```

**What to extract:**
- `announce`: Tracker URL
- `info`: Dictionary containing:
  - `name`: File/folder name
  - `piece length`: Size of each piece
  - `pieces`: SHA-1 hashes of all pieces (20 bytes each)
  - `length`: Total file size (single file mode)
  - `files`: List of files (multi-file mode)

**Additional requirements:**
- Calculate `info_hash` (SHA-1 hash of bencode(info dictionary))
- This `info_hash` uniquely identifies the torrent

**Test:** Load a .torrent file and display all metadata

**Deliverable:** TorrentFile class with parsed metadata

---

## Phase 2: Tracker Communication

### Task 4: Generate Peer ID

**Objective:** Create unique identifier for your client

**Visual: Peer ID Format**
```
┌──────────────────────────────────────┐
│ PEER ID (20 bytes total)             │
├──────────────────────────────────────┤
│                                      │
│  - P C 0 0 0 1 - X X X X X X X X X X │
│  └─┬─┘ └──┬──┘ └────────┬──────────┘│
│    │      │              │           │
│ Client Version      Random chars    │
│  Name   Number         (12 bytes)   │
│                                      │
└──────────────────────────────────────┘

Example: -PC0001-a8f3k2m9x1s4
          └──┬──┘└─────┬──────┘
          Identity  Randomness
```

**Steps:**
- Generate a 20-byte `peer_id` (convention: `-PC0001-` + 12 random characters)
- PC = your client name, 0001 = version

**Deliverable:** Function that generates consistent `peer_id`

---

### Task 5: Implement Tracker Request

**Objective:** Contact the tracker to get list of peers

**Visual: Tracker Communication Flow**
```
    YOUR CLIENT                    TRACKER
         │                            │
         │  HTTP GET Request          │
         │  ?info_hash=...            │
         │  &peer_id=...              │
         │  &port=6881                │
         │  &uploaded=0               │
         │  &downloaded=0             │
         │  &left=2147483648          │
         │  &compact=1                │
         │  &event=started            │
         ├───────────────────────────>│
         │                            │
         │                            │ Look up
         │                            │ peers for
         │                            │ this torrent
         │                            │
         │   Bencode Response         │
         │   {                        │
         │     interval: 1800,        │
         │     peers: <binary>        │
         │   }                        │
         │<───────────────────────────┤
         │                            │
         ↓                            

Parse Peers (Compact Format):
┌────────────────────────────────────┐
│ 6 bytes per peer                   │
├────────────────────────────────────┤
│ [IP byte 1][IP byte 2][IP byte 3]  │
│ [IP byte 4][Port high][Port low]   │
│                                    │
│ Example: [192][168][1][100][0x1A][0xE1] │
│          └───────┬────────┘└───┬───┘   │
│           192.168.1.100     6881       │
└────────────────────────────────────┘
```

**HTTP GET request parameters:**
- `info_hash`: URL-encoded info hash from torrent
- `peer_id`: Your generated peer ID
- `port`: Port you're listening on (use 6881)
- `uploaded`: Bytes uploaded (0 initially)
- `downloaded`: Bytes downloaded (0 initially)
- `left`: Bytes remaining to download
- `compact`: 1 (for compact peer list)
- `event`: 'started' (first request)

**Response parsing:**
- Decode bencode response
- Extract `interval` (how often to update tracker)
- Extract `peers` (binary compact format: 6 bytes per peer = 4 IP + 2 port)

**Test:** Connect to tracker and print list of peer IP addresses

**Deliverable:** TrackerClient class with `get_peers()` method

---

## Phase 3: Peer Wire Protocol

### Task 6: Establish Peer Connection

**Objective:** Connect to peers using TCP sockets

**Visual: Handshake Process**
```
  YOUR CLIENT               PEER
       │                     │
       │  TCP Connect        │
       ├────────────────────>│
       │                     │
       │  Handshake (68 bytes)│
       │  ┌─────────────────┐│
       │  │ 19              ││  1 byte: protocol length
       │  │ "BitTorrent..." ││ 19 bytes: protocol string
       │  │ [00 00 00...]   ││  8 bytes: reserved
       │  │ [info_hash]     ││ 20 bytes: info hash
       │  │ [peer_id]       ││ 20 bytes: peer ID
       │  └─────────────────┘│
       ├────────────────────>│
       │                     │
       │  Peer Handshake     │
       │<────────────────────┤
       │                     │
       │  ✓ Verify info_hash │
       │    matches          │
       │                     │
       ↓                     ↓
   Connected!

Handshake Structure (68 bytes):
┌────┬──────────────┬────────┬──────────┬─────────┐
│ 19 │"BitTorrent..."│00000000│info_hash │peer_id  │
│1 B │   19 bytes   │ 8 bytes│ 20 bytes │20 bytes │
└────┴──────────────┴────────┴──────────┴─────────┘
```

**Steps:**
- Create TCP socket connection to peer
- Send handshake
- Receive and validate peer's handshake
- Verify `info_hash` matches

**Test:** Successfully handshake with at least one peer

**Deliverable:** PeerConnection class with `handshake()` method

---

### Task 7: Implement Message Protocol

**Objective:** Send and receive peer wire protocol messages

**Visual: Message Format & Types**
```
Message Structure:
┌──────────────┬────────────┬─────────────────┐
│ Length (4B)  │ ID (1B)    │ Payload (var)   │
├──────────────┼────────────┼─────────────────┤
│ [00 00 00 05]│ [04]       │ [piece index]   │
└──────────────┴────────────┴─────────────────┘

Message Flow:
    YOU                           PEER
     │                             │
     │  [Interested]               │
     │  ID: 2                      │
     ├────────────────────────────>│
     │                             │
     │         [Unchoke]           │
     │         ID: 1               │
     │<────────────────────────────┤
     │                             │
     │  [Request Block]            │
     │  ID: 6                      │
     ├────────────────────────────>│
     │                             │
     │         [Piece Data]        │
     │         ID: 7               │
     │<────────────────────────────┤
     │                             │

Message Types:
┌────┬──────────────┬──────────────────────────┐
│ ID │ Name         │ Purpose                  │
├────┼──────────────┼──────────────────────────┤
│ 0  │ Choke        │ Stop sending data        │
│ 1  │ Unchoke      │ Can send data            │
│ 2  │ Interested   │ Want peer's data         │
│ 3  │ Not Interest │ Don't want data          │
│ 4  │ Have         │ Announce new piece       │
│ 5  │ Bitfield     │ Pieces available         │
│ 6  │ Request      │ Request block            │
│ 7  │ Piece        │ Block data               │
└────┴──────────────┴──────────────────────────┘
```

**Message types to implement:**
- 0: Choke
- 1: Unchoke
- 2: Interested
- 3: Not interested
- 4: Have (peer has piece)
- 5: Bitfield (which pieces peer has)
- 6: Request (request block of data)
- 7: Piece (block of data)

**Test:** Send interested message and receive unchoke response

**Deliverable:** Methods to send/receive each message type

---

### Task 8: Parse Bitfield

**Objective:** Understand which pieces each peer has

**Visual: Bitfield Representation**
```
Bitfield Message:
┌──────────────────────────────────────────┐
│ Each bit represents one piece           │
├──────────────────────────────────────────┤
│                                          │
│  Byte 1:  [1][0][1][1][0][1][0][0]      │
│  Byte 2:  [1][1][1][0][0][0][1][1]      │
│  Byte 3:  [0][1][0][0][0][0][0][0]      │
│           └───────────┬─────────────┘    │
│              Pieces 0-23                 │
│                                          │
└──────────────────────────────────────────┘

Example for 10 pieces total:
Bytes:  [11010110] [11000000]
         ││││││││   ││
Pieces:  01234567   89

Peer has pieces: 0, 1, 3, 5, 6, 8, 9
Peer missing:     2, 4, 7

Tracking Structure:
┌───────┬────────┬────────┬─────┐
│Piece 0│Piece 1 │Piece 2 │ ... │
├───────┼────────┼────────┼─────┤
│  ✓    │   ✓    │   ✗    │ ... │
│ Has   │  Has   │Missing │     │
└───────┴────────┴────────┴─────┘
```

**Steps:**
- Receive bitfield message after handshake
- Convert bytes to bits (each bit = 1 piece)
- Store which pieces this peer has available

**Deliverable:** Method to track peer's available pieces

---

## Phase 4: Download Logic

### Task 9: Implement Piece Manager

**Objective:** Track download progress and piece integrity

**Visual: Piece Management System**
```
Piece Status Tracking:
┌──────────────────────────────────────────────┐
│ Piece │ Status      │ Hash      │ Peers     │
├───────┼─────────────┼───────────┼───────────┤
│   0   │ ✓ Complete  │ Valid     │ [A,B,C]   │
│   1   │ ⏳ Downloading│ Pending │ [B]       │
│   2   │ ⏳ Downloading│ Pending │ [C]       │
│   3   │ ❌ Missing   │ N/A       │ [A,C]     │
│   4   │ ✓ Complete  │ Valid     │ [A,B,C]   │
│  ...  │             │           │           │
└──────────────────────────────────────────────┘

Rarest First Algorithm:
    Count availability across all peers
              ↓
    ┌────┬────┬────┬────┬────┐
    │P0  │P1  │P2  │P3  │P4  │
    │5   │3   │1   │2   │5   │ ← Peer count
    └────┴────┴────┴────┴────┘
              ↓
    Download P2 first (rarest)
    Then P3, then P1...

Download Flow:
    ┌─────────────┐
    │ Get Next    │
    │ Piece       │
    └──────┬──────┘
           ↓
    ┌─────────────┐
    │ Download    │
    │ Blocks      │
    └──────┬──────┘
           ↓
    ┌─────────────┐
    │ Verify      │
    │ SHA-1 Hash  │
    └──────┬──────┘
           ↓
    ┌─────────────┐     ✓ Valid
    │ Complete?   ├──────────> ✅ Mark Complete
    └──────┬──────┘
           │ ✗ Invalid
           ↓
    ❌ Redownload
```

**Data structures needed:**
- List of all pieces with SHA-1 hashes
- Status for each piece (missing/downloading/complete)
- Map of which peers have which pieces

**Key methods:**
- `get_next_piece()`: Select next piece to download
- `verify_piece()`: Check SHA-1 hash of downloaded piece
- `mark_complete()`: Mark piece as done

**Strategy:** Start with rarest piece first (rarest first algorithm)

**Deliverable:** PieceManager class tracking all pieces

---

### Task 10: Request Blocks from Peers

**Objective:** Download pieces in 16KB blocks

**Visual: Block-Based Downloading**
```
Why Blocks?
┌────────────────────────────────────────┐
│ One Piece (256 KB)                     │
├────────────────────────────────────────┤
│ Too large to send at once!             │
│                                        │
│ Split into Blocks (16 KB each):       │
│ ┌─────┬─────┬─────┬─────┬─────┬───┐  │
│ │Blk 0│Blk 1│Blk 2│Blk 3│ ... │15 │  │
│ │16KB │16KB │16KB │16KB │     │16K│  │
│ └─────┴─────┴─────┴─────┴─────┴───┘  │
│   ↓     ↓     ↓     ↓     ↓     ↓    │
│ Download separately, reassemble       │
└────────────────────────────────────────┘

Request Message Format:
┌──────────────┬──────────────┬─────────────┐
│ Piece Index  │ Block Offset │ Block Length│
│   (4 bytes)  │  (4 bytes)   │  (4 bytes)  │
├──────────────┼──────────────┼─────────────┤
│      3       │      0       │   16384     │  Block 0
│      3       │   16384      │   16384     │  Block 1
│      3       │   32768      │   16384     │  Block 2
└──────────────┴──────────────┴─────────────┘

Pipelining (5-10 requests):
    YOU                           PEER
     │                             │
     │  Request Block 0            │
     ├────────────────────────────>│
     │  Request Block 1            │
     ├────────────────────────────>│
     │  Request Block 2            │
     ├────────────────────────────>│
     │         ...                 │
     │                             │
     │      Block 0 Data           │
     │<────────────────────────────┤
     │  Request Block 3  (refill)  │
     ├────────────────────────────>│
     │      Block 1 Data           │
     │<────────────────────────────┤
     │  Request Block 4  (refill)  │
     ├────────────────────────────>│
     │         ...                 │
```

**Why blocks?**
- Pieces can be large (256KB-1MB typical)
- Download in 16KB chunks for efficiency

**Request message format:**
- Piece index (4 bytes)
- Block offset within piece (4 bytes)
- Block length (4 bytes) - typically 16KB

**Steps:**
- Calculate how many blocks per piece
- Send multiple requests to keep pipeline full (5-10 requests)
- Handle received piece messages

**Test:** Download one complete piece and verify hash

**Deliverable:** Method to request and receive blocks

---

### Task 11: Implement Download Manager

**Objective:** Coordinate downloading from multiple peers

**Visual: Multi-Peer Download Orchestration**
```
Download Manager Overview:
┌──────────────────────────────────────────┐
│         DOWNLOAD MANAGER                 │
├──────────────────────────────────────────┤
│                                          │
│  ┌────────┐  ┌────────┐  ┌────────┐    │
│  │ Peer A │  │ Peer B │  │ Peer C │    │
│  └───┬────┘  └───┬────┘  └───┬────┘    │
│      │           │           │          │
│      ↓           ↓           ↓          │
│  ┌──────────────────────────────────┐  │
│  │   PIECE ASSIGNMENT               │  │
│  ├──────────────────────────────────┤  │
│  │ Peer A → Piece 0, 3, 6           │  │
│  │ Peer B → Piece 1, 4, 7           │  │
│  │ Peer C → Piece 2, 5, 8           │  │
│  └──────────────────────────────────┘  │
│                                          │
│  ┌──────────────────────────────────┐  │
│  │   PIPELINE PER PEER              │  │
│  ├──────────────────────────────────┤  │
│  │ Peer A: [Req] [Req] [Req] [Req]  │  │
│  │ Peer B: [Req] [Req] [Req] [Req]  │  │
│  │ Peer C: [Req] [Req] [Req] [Req]  │  │
│  └──────────────────────────────────┘  │
└──────────────────────────────────────────┘

Connection Management:
    ┌──────────┐
    │ Peer A   │ ─────> ✓ Connected
    ├──────────┤
    │ Peer B   │ ─────> ✓ Connected
    ├──────────┤
    │ Peer C   │ ─────> ⏳ Connecting
    ├──────────┤
    │ Peer D   │ ─────> ❌ Disconnected
    └──────────┘              ↓
                         Reassign pieces

Timeline:
    t=0s   [===A===][===B===][===C===]
           P0       P1       P2
    
    t=5s   [===A===][===B===][===C===]
           P3       P4       P5
    
    t=10s  [===A===][===B===][=C drops=]
           P6       P7       ↓
                            Reassign P8→A
```

**Requirements:**
- Manage connections to multiple peers (3-5 peers)
- Distribute piece requests across peers
- Handle peer disconnections
- Reassign pieces if peer drops

**Pipeline management:**
- Keep 5-10 outstanding requests per peer
- Request next block immediately when one completes

**Test:** Download entire file from multiple peers

**Deliverable:** DownloadManager class orchestrating downloads

---

### Task 12: Write Pieces to Disk

**Objective:** Assemble and save the downloaded file

**Visual: Disk Writing Process**
```
Single File Mode:
┌────────────────────────────────────────────┐
│          OUTPUT FILE                       │
├────────────────────────────────────────────┤
│                                            │
│  Piece 2 arrives first ────> Write at offset 524288
│  ┌────┬────┬────┬────┬────┬────┐         │
│  │ ?? │ ?? │ P2 │ ?? │ ?? │ ?? │         │
│  └────┴────┴────┴────┴────┴────┘         │
│                                            │
│  Piece 0 arrives      ────> Write at offset 0
│  ┌────┬────┬────┬────┬────┬────┐         │
│  │ P0 │ ?? │ P2 │ ?? │ ?? │ ?? │         │
│  └────┴────┴────┴────┴────┴────┘         │
│                                            │
│  Fill in remaining pieces...              │
│  ┌────┬────┬────┬────┬────┬────┐         │
│  │ P0 │ P1 │ P2 │ P3 │ P4 │ P5 │         │
│  └────┴────┴────┴────┴────┴────┘         │
│                Complete! ✓                 │
└────────────────────────────────────────────┘

Multi-File Mode:
┌────────────────────────────────────────────┐
│ Torrent spans multiple files:              │
│                                            │
│  Piece 0: [=============================]  │
│           ↓                         ↓      │
│      file1.txt              file2.txt      │
│      (bytes 0-200K)         (bytes 200K-) │
│                                            │
│  Directory Structure:                      │
│  folder/                                   │
│    ├── file1.txt                          │
│    ├── file2.txt                          │
│    └── subfolder/                         │
│        └── file3.txt                      │
│                                            │
└────────────────────────────────────────────┘

Offset Calculation:
    Piece Index × Piece Length = File Offset
    
    Example:
    Piece 3 × 256KB = 768KB offset in file
```

**Single file mode:**
- Write pieces to file at correct offsets
- Handle pieces arriving out of order

**Multi-file mode:**
- Calculate which bytes belong to which file
- Create directory structure
- Write to multiple files

**Test:** Download complete file and verify it matches original

**Deliverable:** Method to write completed pieces to disk

---

## Phase 5: Polish & Features

### Task 13: Add Progress Display

**Objective:** Show download statistics

**Visual: Progress Display Layout**
```
┌────────────────────────────────────────────┐
│ 🌐 BitTorrent Client                       │
├────────────────────────────────────────────┤
│                                            │
│ File: ubuntu-20.04.iso                     │
│ Size: 2.5 GB                               │
│                                            │
│ Progress: [████████████░░░░] 65.3%        │
│                                            │
│ ⬇️  Download:  15.4 MB/s                   │
│ ⬆️  Upload:     2.1 MB/s                   │
│                                            │
│ Pieces: 6,530 / 10,000 complete            │
│                                            │
│ Peers: 🟢🟢🟢🔴🔴 (3/5 connected)           │
│   • 192.168.1.100:6881  [Active]          │
│   • 10.0.0.54:51234     [Active]          │
│   • 172.16.0.88:6889    [Active]          │
│                                            │
│ ⏱️  ETA: 2m 35s                             │
│                                            │
└────────────────────────────────────────────┘

Real-time Updates (every 1 second):
    t=0s:   [██░░░░░░░░] 20% | 10 MB/s
    t=1s:   [███░░░░░░░] 30% | 12 MB/s
    t=2s:   [████░░░░░░] 40% | 15 MB/s
```

**Display:**
- Download speed (KB/s)
- Progress percentage
- Pieces completed/total
- Connected peers
- Time remaining estimate

**Deliverable:** Real-time progress updates in terminal

---

### Task 14: Implement Resume Capability

**Objective:** Resume interrupted downloads

**Visual: Resume Process**
```
Initial Download:
┌────┬────┬────┬────┬────┬────┐
│ ✓  │ ✓  │ ⏳ │ ❌ │ ❌ │ ❌ │  Pieces
└────┴────┴────┴────┴────┴────┘
  P0   P1   P2   P3   P4   P5
         ↓
    Connection Lost! ⚡
         ↓
Save State File:
┌──────────────────────┐
│ state.json           │
├──────────────────────┤
│ {                    │
│   "completed": [0,1] │
│   "total": 6         │
│   "file": "data.bin" │
│ }                    │
└──────────────────────┘

Resume Download:
         ↓
┌────┬────┬────┬────┬────┬────┐
│ ✓  │ ✓  │ ⏳ │ ⏳ │ ⏳ │ ⏳ │  Resume from P2
└────┴────┴────┴────┴────┴────┘
 Skip  Skip   Download remaining
  P0    P1

Verification on Resume:
┌─────────────────────┐
│ 1. Load state file  │
├─────────────────────┤
│ 2. Read disk pieces │
├─────────────────────┤
│ 3. Verify hashes    │ ← Important!
├─────────────────────┤
│ 4. Resume download  │
└─────────────────────┘
```

**Steps:**
- Save state file with completed pieces
- On restart, skip downloading completed pieces
- Verify existing pieces on disk

**Deliverable:** Ability to pause and resume downloads

---

### Task 15: Error Handling & Edge Cases

**Objective:** Make client robust

**Visual: Error Scenarios**
```
Error Handling Flow:
┌──────────────────────────────────────┐
│ ERROR TYPE         │ RESPONSE        │
├────────────────────┼─────────────────┤
│                    │                 │
│ Peer Disconnect    │ ┌─────────────┐│
│      ⚡            │ │ Reconnect   ││
│                    │ │ Reassign    ││
│                    │ │ pieces      ││
│                    │ └─────────────┘│
│                    │                 │
│ Invalid Hash       │ ┌─────────────┐│
│      ✗             │ │ Discard     ││
│                    │ │ piece       ││
│                    │ │ Redownload  ││
│                    │ └─────────────┘│
│                    │                 │
│ Tracker Timeout    │ ┌─────────────┐│
│      ⏱️             │ │ Retry with  ││
│                    │ │ backoff     ││
│                    │ └─────────────┘│
│                    │                 │
│ Malformed Message  │ ┌─────────────┐│
│      ❌            │ │ Log error   ││
│                    │ │ Close conn  ││
│                    │ └─────────────┘│
│                    │                 │
└──────────────────────────────────────┘

Retry Logic:
    Attempt 1  ────> Fail
         ↓
    Wait 1s
         ↓
    Attempt 2  ────> Fail
         ↓
    Wait 2s
         ↓
    Attempt 3  ────> Success ✓

Try-Catch Structure:
    ┌─────────────┐
    │ Try         │
    │  Download   │
    └──────┬──────┘
           │
    ┌──────↓──────┐
    │ Exception?  │
    └──────┬──────┘
           │
    ┌──────↓──────┐
    │ Log Error   │
    │ Retry/Skip  │
    └─────────────┘
```

**Handle:**
- Peer disconnections mid-download
- Invalid piece hashes (redownload)
- Tracker connection failures
- Timeout scenarios
- Malformed messages

**Deliverable:** Stable client that handles errors gracefully

---

### Task 16: Testing & Documentation

**Objective:** Validate and document your implementation

**Visual: Testing Strategy**
```
Test Pyramid:
         ┌─────────┐
         │E2E Tests│         Full downloads
         └─────────┘
       ┌─────────────┐
       │Integration  │       Multi-component
       │   Tests     │       interactions
       └─────────────┘
    ┌──────────────────┐
    │   Unit Tests     │    Individual functions
    │                  │    bencode, hashing, etc.
    └──────────────────┘

Test Cases:
┌────────────────────────────────────────┐
│ ✓ Parse valid .torrent file           │
│ ✓ Parse corrupted .torrent file       │
│ ✓ Connect to tracker                  │
│ ✓ Handshake with peer                 │
│ ✓ Download single-file torrent        │
│ ✓ Download multi-file torrent         │
│ ✓ Verify piece integrity              │
│ ✓ Handle peer disconnection           │
│ ✓ Resume interrupted download         │
│ ✓ Multi-peer coordination             │
└────────────────────────────────────────┘

Documentation Structure:
    README.md
    ├── Installation
    ├── Usage
    ├── Architecture
    └── Examples
    
    ARCHITECTURE.md
    ├── Component Diagram
    ├── Data Flow
    └── Protocol Details
    
    API_DOCS.md
    ├── Class Reference
    ├── Function Reference
    └── Examples
```

**Testing:**
- Test with various torrent files
- Test with single-file and multi-file torrents
- Verify downloaded files match originals
- Test with different numbers of peers

**Documentation:**
- Code comments explaining protocol details
- README with usage instructions
- Architecture diagram
- Performance metrics

**Deliverable:** Complete documentation package

---

## Phase 6: Thesis Write-up

### Task 17: Write Thesis Document

**Visual: Thesis Structure**
```
┌────────────────────────────────────────────┐
│         THESIS OUTLINE                     │
├────────────────────────────────────────────┤
│                                            │
│ 1. INTRODUCTION                            │
│    ├── Problem Statement                   │
│    ├── Objectives                          │
│    └── Structure Overview                  │
│                                            │
│ 2. TECHNICAL BACKGROUND                    │
│    ├── P2P Architecture                    │
│    │   [Diagram: Centralized vs P2P]       │
│    ├── BitTorrent Protocol                 │
│    │   [Flow: Tracker → Peers → Data]      │
│    └── Related Work                        │
│                                            │
│ 3. IMPLEMENTATION                          │
│    ├── Architecture Design                 │
│    │   [Component Diagram]                 │
│    ├── Key Components                      │
│    │   • Torrent Parser                    │
│    │   • Tracker Client                    │
│    │   • Peer Manager                      │
│    │   • Download Engine                   │
│    └── Algorithms                          │
│        [Flowchart: Piece Selection]        │
│                                            │
│ 4. RESULTS & ANALYSIS                      │
│    ├── Performance Metrics                 │
│    │   [Graph: Download Speed vs Peers]    │
│    ├── Comparison                          │
│    │   [Table: vs Standard Clients]        │
│    └── Challenges                          │
│                                            │
│ 5. CONCLUSION                              │
│    ├── Summary                             │
│    ├── Future Work                         │
│    └── Applications                        │
│                                            │
└────────────────────────────────────────────┘

Key Diagrams to Include:
┌─────────────────────────┐
│ System Architecture     │
│                         │
│  [Tracker]              │
│      ↕                  │
│  [Your Client]          │
│   ↙  ↓  ↘              │
│ [P1][P2][P3]            │
└─────────────────────────┘

┌─────────────────────────┐
│ Download Performance    │
│                         │
│ Speed                   │
│   ↑                     │
│   │     ╱──────         │
│   │   ╱                 │
│   │ ╱                   │
│   └──────────> Peers    │
└─────────────────────────┘
```

**Sections to include:**

**1. Introduction**
- BitTorrent protocol overview
- Project motivation and goals

**2. Technical Background**
- P2P architecture
- BitTorrent protocol specification
- Bencode format
- Tracker protocol
- Peer wire protocol

**3. Implementation**
- Architecture design
- Component descriptions
- Key algorithms (piece selection, etc.)
- Code examples

**4. Results & Analysis**
- Performance measurements
- Download speeds achieved
- Comparison with standard clients
- Challenges encountered

**5. Conclusion**
- Lessons learned
- Future improvements
- Applications of P2P concepts

**Deliverable:** Complete thesis document

---

## Technical Resources

### Key Specifications:
- **BEP 0003:** The BitTorrent Protocol Specification
- **BEP 0023:** Tracker Returns Compact Peer Lists

### Helpful Concepts:
```
┌────────────────────────────────────────┐
│ TERMINOLOGY                            │
├────────────────────────────────────────┤
│ • Info Hash:  Unique torrent ID        │
│ • Piece:      Large file chunk         │
│               (256KB-1MB)              │
│ • Block:      Piece subdivision (16KB) │
│ • Choke:      Stop data flow           │
│ • Unchoke:    Allow data flow          │
│ • Seeder:     Has complete file        │
│ • Leecher:    Downloading file         │
└────────────────────────────────────────┘
```

### Python Libraries:
- `socket`: TCP connections
- `struct`: Binary data packing/unpacking
- `hashlib`: SHA-1 hashing
- `requests`: HTTP tracker requests

---

## Testing Recommendations

```
┌────────────────────────────────────────┐
│ TESTING BEST PRACTICES                 │
├────────────────────────────────────────┤
│                                        │
│ 1. Start Small                         │
│    └─> 1-10 MB test files             │
│                                        │
│ 2. Use Legal Content                   │
│    └─> Linux ISOs, open-source        │
│                                        │
│ 3. Log Everything                      │
│    └─> Debug = find bugs fast         │
│                                        │
│ 4. Test Incrementally                  │
│    └─> Verify each phase works        │
│                                        │
│ 5. Compare Protocols                   │
│    └─> Use Wireshark to inspect       │
│                                        │
└────────────────────────────────────────┘
```

---

## Expected Timeline

```
Weeks 1-2   [████████░░░░░░░░░░░░░░]  Tasks 1-3   Parsing
Week 3      [████████████░░░░░░░░░░]  Tasks 4-5   Tracker
Weeks 4-5   [████████████████░░░░░░]  Tasks 6-8   Peer Protocol
Weeks 6-8   [████████████████████░░]  Tasks 9-12  Download Logic
Week 9      [██████████████████████]  Tasks 13-15 Polish
Week 10     [██████████████████████]  Task 16     Testing
Weeks 11-12 [██████████████████████]  Task 17     Thesis
            └────────────────────────┘
              0%              100%
```

---

## Success Criteria

```
✅ CHECKLIST:
┌────────────────────────────────────────┐
│ ✓ Parse any valid .torrent file       │
│ ✓ Contact tracker and retrieve peers  │
│ ✓ Connect to multiple peers            │
│ ✓ Download file pieces from peers      │
│ ✓ Verify piece integrity (SHA-1)       │
│ ✓ Assemble complete file correctly      │
│ ✓ Handle single & multi-file torrents   │
│ ✓ Display download progress             │
│ ✓ Gracefully handle errors              │
└────────────────────────────────────────┘
```

**Good luck with your implementation! 🚀**
