---
layout: post
title: "Reading a Native Backup Through --debug"
date: 2026-03-16
author: loichrn
categories: backup architecture
description: >-
  A short walkthrough of Cloudstic debug output: repository initialization,
  first Google Drive backup, incremental scans through change tokens, and the
  role of packfiles.
tags: [backup, debug, google-drive, cloudstic, incremental, packfiles]
---

When I build this kind of tool, I like to look at the debug output.

It is the easiest way to see if the architecture really does what it claims.

In the case of a native Google Drive backup, I want to verify a few things:

- the repository is initialized cleanly
- the first backup performs a full scan
- the next backup does not scan the whole drive again
- small objects are not written one by one to the object store

If you want the general explanation first, you can read [Google Drive Is Not a Backup (Here's How to Fix That)](/2026/03/04/google-drive-is-not-a-backup-heres-how-to-fix-that/) and [Why Sync Is Not Backup, Technically](/2026/03/16/why-sync-is-not-backup-technically/).

This article is only about the logs.

## Prerequisites

Before reproducing the debug runs below, make sure you have:

- the Cloudstic CLI installed by following the [installation guide](https://docs.cloudstic.com/installation)
- a repository destination configured and reachable
- Google Drive authentication available for the `gdrive-changes` source
- `--debug` output enabled on a test setup rather than on production data you do not want to inspect live

## Repository Initialization

Here is a representative `cloudstic init --debug` run:

```text
$ cloudstic init --debug
[store #1] GET    config                                              2074.6ms err=NoSuchKey
[store #2] LIST   keys/                                                 99.4ms
[store #3] PUT    keys/kms-platform-default                            119.8ms 311B
[store #4] PUT    config                                               123.6ms 63B
Created new encryption key slots.
Repository initialized (encrypted: true).
```

This is short, but it already tells a lot.

`GET config` confirms that the repository does not exist yet.

`LIST keys/` checks the existing key slots.

`PUT keys/...` creates the encryption slot.

`PUT config` writes the repository marker.

So before backing up any file, the repository has an identity and key material.

## First Backup

Now the first Google Drive backup:

```text
$ cloudstic backup -source gdrive-changes --debug
[store #8] GET    index/snapshots                                      101.0ms err=NoSuchKey
[hamt] get node/... hit staging (158 bytes)
...
Scanning             ... done! [20 in 790ms]
[store #14] PUT    chunk/d2667...                                      807.7ms 1.2MB
[store #15] PUT    chunk/3134f...                                      261.2ms 587.8KB
...
Uploading            ... done! [45.65MB in 5.995s]
[store #51] PUT    packs/d7596...                                      191.9ms 1.2MB
Backup complete. Snapshot: snapshot/6f70aa...
```

The main points are simple.

`GET index/snapshots` returns `NoSuchKey`, so there is no previous snapshot.

This means the source has to perform a full scan. This is expected, even with `gdrive-changes`.

`PUT chunk/...` means new file data is uploaded.

`PUT packs/...` means Cloudstic is bundling small metadata objects into packfiles instead of writing thousands of tiny objects individually.

That point is important for object stores like S3. It reduces API calls and makes metadata reads cheaper on later runs.

## What the First Snapshot Gives You

After this first backup, Cloudstic has more than just stored bytes.

It has created a real snapshot containing:

- references to data chunks
- metadata nodes
- source identity
- backup metadata
- for an incremental source, the change token for the next run

This is what makes the second backup interesting.

## Second Backup

Here is a representative second run with no changes:

```text
$ cloudstic backup -source gdrive-changes --debug
[store #8] GET    index/snapshots                                      115.8ms 350B
[store #10] GET   packs/d7596...                                       729.8ms 1.2MB
Scanning (incremental) ... done! [0 in 212ms]
...
Added to the repository: 286 B (315 B compressed)
Processed 0 entries in 1s
Snapshot 3eb699... saved
```

This is the part I care about most.

`Scanning (incremental) ... done! [0 in 212ms]`

This means Cloudstic reused the previous snapshot, read the stored change token, asked Google Drive for the delta, and found nothing to process.

So it did not walk the whole tree again.

This is the practical difference between a native integration and a mounted drive scanned like a local filesystem.

## Why `GET packs/...` Is Good News

On the second run, there is still a metadata read:

```text
[store #10] GET   packs/d7596...                                       729.8ms 1.2MB
```

This is normal.

The engine still needs metadata to reason about the previous state.

The good sign is that it fetches a packfile, not hundreds of tiny metadata objects.

This is exactly why packfiles exist.

## Why a No-Change Backup Still Writes a Few Bytes

This line is also expected:

```text
Added to the repository: 286 B (315 B compressed)
```

Even if the files did not change, the backup still produces a new checkpoint.

So Cloudstic writes a very small amount of metadata for the new snapshot and the updated incremental state.

That is what you want: a new restore point without re-uploading the source.

## What the Logs Show

For me, the logs validate four things:

- the first run is a normal full backup
- the second run uses Google Drive's native change tracking
- the repository stores immutable snapshots
- packfiles avoid object-store overhead on small metadata

This is why the second backup can complete in about a second.

It is not doing the same work again. It is using the source's own incremental mechanism and reusing the existing structure.

That is also why I find `--debug` useful. It lets you see the difference between a real native backup flow and a tool that only looks native from the outside.

If you want to try this yourself, start with the [Cloudstic installation guide](https://docs.cloudstic.com/installation).
