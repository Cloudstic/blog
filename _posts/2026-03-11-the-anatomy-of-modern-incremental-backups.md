---
layout: post
title: "The Anatomy of Modern Incremental Backups"
date: 2026-03-11
author: loichrn
categories: backup architecture
description: >-
  A deep dive into content-addressable storage, content-defined chunking,
  and Merkle trees — the building blocks of modern incremental backup engines.
tags: [backup, incremental, deduplication, merkle-tree, content-addressable-storage, encryption]
---

If you have ever tried to back up a terabyte of data to the cloud, you know the naive approach of copying everything every time is a disaster. It is slow, expensive, and kills your bandwidth. To solve this, the industry moved to incremental backups where only changed data is uploaded. However, as data scales into the petabytes, the standard way of doing incrementals is hitting a wall.

In this article, we look at the state of the art techniques used by modern storage engines and why the next generation of backup tech is moving toward a more flexible, flat architecture.

## 1. Content Addressable Storage (CAS)

The biggest leap in modern backup architecture is abandoning files and folders at the storage layer. Instead, industry leaders use **Content Addressable Storage (CAS)**. In a traditional system, data is found by its path. In CAS, data is found by its cryptographic hash such as SHA-256.

```mermaid
graph LR
    subgraph "Traditional: Location-Based"
        P1["/docs/report.pdf"] -->|path lookup| D1["Data Block A"]
        P2["/backup/report.pdf"] -->|path lookup| D2["Data Block A (duplicate)"]
    end

    subgraph "CAS: Content-Based"
        F1["report.pdf"] -->|SHA-256| H1["a8f5e..."]
        F2["report_copy.pdf"] -->|SHA-256| H1
        H1 -->|hash lookup| D3["Data Block A (stored once)"]
    end
```

The CAS logic is simple. Same content leads to the same hash which is stored once. This is the foundation of **deduplication**. If you rename a file or move it to a different folder, a CAS based system realizes the content has not changed and uploads exactly zero bytes.

### The Confirmation Attack

Standard CAS has a privacy hole. If a cloud provider knows the hash of a specific file, they can see if your bucket contains that hash even if they cannot read the encrypted content.

To solve this, one solution is to avoid using raw hashes for chunk names. Instead, we can use **Keyed HMAC-SHA256**. By using a secret key known only to you to salt the hashes, your data remains fully deduplicated but becomes cryptographically opaque to the host.

### The Boundary Shift Problem

What if you have a 10GB database and change one single row?

Slicing it at fixed 1MB intervals is a trap. If you insert one byte at the very beginning, every 1MB boundary shifts. This changes the hash of every block.

### The Solution: Content Defined Chunking (CDC)

Modern systems use algorithms like **FastCDC**. It rolls a sliding window across the data. When the data matches a specific mathematical pattern, it slices the chunk.

```mermaid
graph TD
    subgraph "Fixed-Size Chunking"
        FS["Original File: AABBCCDDEE"]
        FS --> C1["AA | BB | CC | DD | EE"]
        INS1["Insert 'X' at start: XAABBCCDDEE"]
        INS1 --> C2["XA | AB | BC | CD | DE"]
        C2 -.- NOTE1["⚠ Every chunk changed!"]
    end

    subgraph "Content-Defined Chunking (CDC)"
        CDC["Original File: AABBCCDDEE"]
        CDC --> C3["AAB | BCC | DDEE"]
        INS2["Insert 'X' at start: XAABBCCDDEE"]
        INS2 --> C4["XAAB | BCC | DDEE"]
        C4 -.- NOTE2["✓ Only first chunk changed"]
    end
```

Because the boundaries are determined by the content itself, an insertion at the start only changes the first chunk. The rest of the puzzle pieces realign perfectly.

## 2. Structural Sharing: The Magic of Merkle Trees

Once you have thousands of content addressed chunks, you must remember which chunks belong to which files at a specific point in time. The industry standard is the **Merkle Tree**. This is a hierarchical structure where every node is the hash of its children. This enables **Structural Sharing**.

```mermaid
graph TD
    subgraph "Snapshot 1"
        R1["Root Hash: abc123"] --> A1["Dir A: def456"]
        R1 --> B1["Dir B: 789abc"]
        A1 --> F1["file1.txt: aaa"]
        A1 --> F2["file2.txt: bbb"]
        B1 --> F3["file3.txt: ccc"]
    end

    subgraph "Snapshot 2 (file2.txt changed)"
        R2["Root Hash: xyz789"] --> A2["Dir A: new111"]
        R2 -.->|"shared"| B1
        A2 -.->|"shared"| F1
        A2 --> F2new["file2.txt: ddd (new)"]
    end

    style B1 fill:#90EE90
    style F1 fill:#90EE90
    style F3 fill:#90EE90
    style F2new fill:#FFB347
```

When you take a new backup:

- You only upload the **modified chunks**.
- A new root hash is generated for the snapshot.
- Most of the tree simply **points to the previous snapshot's nodes**.

The result is that every snapshot acts like a full backup for recovery, but it only consumes the storage space of an incremental update.

## 3. The Directory Tree Trap

Most modern tools build their Merkle Trees by mirroring the computer's natural directory structure. While intuitive, this path based approach creates two major headaches at scale.

**Path Amplification:** If you have a file buried ten folders deep, changing that one file forces the system to recalculate and reupload metadata for all ten parent directories.

**The Rigid Hierarchy Problem:** Modern data does not always live in a neat tree. Cloud drives like Google Drive allow a single file to have multiple parents, and flat data streams like database dumps do not have folders at all. Standard tools struggle to map these non tree structures efficiently, which often leads to metadata bloat.

## 4. About the Author

I have been looking into these industry standards while building [Cloudstic](https://github.com/Cloudstic), a project aimed at solving these specific bottlenecks. To achieve both zero trust privacy and high performance, I realized the engine needed to be built from the ground up.

Cloudstic is a content addressable, encrypted CLI backup tool designed to be fully stateless. Unlike many traditional tools, it can back up from Google Drive and OneDrive using incremental APIs without ever needing to mount the drive.

**Core Features:**

- **Encrypted by Default:** Uses AES-256-GCM encryption with password, platform key, or recovery key slots.
- **Content Addressable:** Native deduplication across all sources; identical files are stored only once.
- **Versatile Backends:** Supports local filesystems, Amazon S3 (R2, MinIO), and Backblaze B2.
- **Smart Retention:** Automated keep-last, hourly, daily, and yearly retention policies.
- **Point-in-Time Restore:** Instant access to any snapshot or file from any point in history.
