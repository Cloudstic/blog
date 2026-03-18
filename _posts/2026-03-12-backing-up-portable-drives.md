---
layout: post
title: "Backing Up Portable Drives"
date: 2026-03-12
author: loichrn
categories: backup storage
description: >-
  Portable drives move between machines, mount points, and operating systems.
  This article explains why many backup tools break incremental history in that
  situation, and how Cloudstic uses GPT partition UUIDs to keep a single backup
  lineage across machines.
tags: [backup, portable-drive, external-drive, incremental, restic, borg, uuid, cross-machine]
---

A portable drive is convenient storage but It is not a backup.

There is also a second problem that appears as soon as you try to back it up seriously: the drive moves.

On one machine it is mounted as `/Volumes/WorkDrive`.

On another one it becomes `/media/alice/WorkDrive`.

On Windows it may become `E:\`.

For a human, this is obviously the same drive, for many backup tools, it is not.

## The Actual Problem

Many backup tools identify a source using some combination of:

- hostname
- absolute path
- mount point

This is acceptable for internal disks.

It is not acceptable for portable drives.

If the identity changes every time the drive moves to another machine, incremental history breaks. Even when deduplication prevents re-uploading all data, the tool still has to rebuild state as if it had discovered a new source.

In practice, this means more scanning, more metadata, and a fragmented backup history.

## What the Identity Should Be

For a portable drive, the identity should come from the drive itself.

On modern drives, the right identifier already exists: the GPT partition UUID.

This UUID is:

- stable across reboots
- independent of mount point
- independent of hostname
- usable across operating systems

This is the identifier I wanted Cloudstic to use for portable drives.

## How Common Tools Behave

There are several good backup tools on the market. The issue here is not quality in general. The issue is portable-drive identity.

### rsync

`rsync` is a very useful file copy tool.

It can be used to build snapshot-like workflows with `--link-dest`, but it does not have a real notion of encrypted repository, snapshot identity, or portable drive UUID.

For this use case, it requires scripting and discipline.

### Time Machine

Time Machine is tied to macOS and works per machine.

If the same portable drive is backed up from two different Macs, you effectively get two different histories.

### Restic and Borg

Restic and Borg are serious backup tools, and I like both.

However, their default source identity model is still centered on the machine and the path.

You can work around this with manual overrides such as `--host`, but then the correctness of the history depends on the user remembering to keep the override identical everywhere.

This works, but I do not consider it a satisfying solution for a portable drive.

## The Approach in Cloudstic

Cloudstic treats a portable local source differently.

When a volume UUID is available, it takes precedence over hostname and path for snapshot matching.

This means that if you:

1. back up a drive on macOS
2. unplug it
3. connect it to Linux
4. run the backup again

Cloudstic can continue the same snapshot lineage automatically.

This is the behavior I expected from the start.

The path inside the snapshot is stored relative to the volume root, so changing the mount point does not create an artificial identity change.

## What This Changes in Practice

Suppose you have a 100 GB external SSD.

You back it up from machine A.

The next day you modify only 200 MB of files and plug the same drive into machine B.

The correct outcome is simple: the next backup should process the same source lineage and upload only the delta.

That is exactly what a UUID-based identity gives.

Without it, you still get deduplication benefits, but you lose the continuity of the source history.

## Prerequisites

Before running the examples below, make sure you have:

- the Cloudstic CLI installed on each machine that will back up the drive by following the [installation guide](https://docs.cloudstic.com/installation)
- a portable drive formatted with a stable partition identifier such as a GPT partition UUID
- access to the backup repository from every machine involved
- repository credentials available before you start the first backup

## Minimal Example with Cloudstic

### 1. Initialize the Repository

For example on local storage:

```bash
cloudstic init \
  -store local:/Volumes/BackupDrive/cloudstic \
  -recovery
```

Or on object storage, for example S3-compatible storage:

```bash
export CLOUDSTIC_STORE=s3:my-backup-bucket
export AWS_ACCESS_KEY_ID=your-key
export AWS_SECRET_ACCESS_KEY=your-secret

cloudstic init -recovery
```

### 2. First Backup on Machine A

```bash
cloudstic backup \
  -source local:/Volumes/WorkDrive \
  -exclude ".Spotlight-V100/" \
  -exclude ".fseventsd/" \
  -exclude ".Trashes/" \
  -exclude ".DS_Store" \
  -tag work-drive
```

At this point Cloudstic detects the volume UUID and stores it in the snapshot metadata.

### 3. Second Backup on Machine B

```bash
cloudstic backup \
  -source local:/media/alice/WorkDrive \
  -exclude ".Spotlight-V100/" \
  -exclude ".fseventsd/" \
  -exclude ".Trashes/" \
  -tag work-drive
```

Same drive, different hostname, different mount point.

Cloudstic matches the volume UUID and continues the same incremental chain.

On macOS and Linux this UUID is detected automatically. When auto-detection is unavailable, you can still provide `-volume-uuid` manually.

## Retention Also Becomes Cleaner

This matters for retention as well.

If snapshots are grouped by volume UUID, a retention policy applies to the drive as one source, not once per machine.

So keeping 7 daily snapshots means 7 daily snapshots for the portable drive, not 7 on the Mac side and 7 on the Linux side.

This is a small detail, but it keeps history much cleaner.

## Conclusion

Portable drives move between machines.

If the backup tool identifies them by hostname or mount point, it will eventually treat the same drive as multiple sources.

For me, this is the wrong model.

The identity should come from the drive itself.

Using the GPT partition UUID solves that problem and makes cross-machine incremental backups behave as expected.

That is the approach implemented in Cloudstic for portable local sources.

If you want to get started, see the [installation guide](https://docs.cloudstic.com/installation). If you want the full command reference, see the [Cloudstic CLI documentation](https://docs.cloudstic.com).
