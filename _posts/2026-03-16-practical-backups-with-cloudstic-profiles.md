---
layout: post
title: "Practical Backups with Cloudstic Profiles"
date: 2026-03-16
author: loichrn
categories: backup cli
description: >-
  Learn how to turn Cloudstic CLI into a practical daily backup workflow using
  profiles, named stores, reusable auth entries, and secret references instead
  of fragile shell history and copied command lines.
tags: [backup, cli, profiles, secrets, s3, google-drive, automation]
---

The first backup command is easy.

The fiftieth backup command is where things usually fall apart.

At the beginning, most people run something like this:

```bash
cloudstic backup \
  -store s3:my-backups/cloudstic \
  -password "correct horse battery staple" \
  -source local:~/Documents
```

It works. It is also how you end up with secrets in shell history, copy-pasted commands in notes, and one slightly different command per machine.

The better pattern is to move from one-off commands to **profiles**.

Profiles let you define:

- what to back up
- where it should go
- which auth entry to use for cloud sources
- which secret references should be resolved at runtime

In other words: fewer flags, fewer mistakes, and something you can automate without fear.

This article walks through a practical setup for everyday use.

- We will create a named store.
- We will create a few backup profiles.
- We will store secrets safely.
- We will run backups with short, repeatable commands.

If you want the full reference while following along, see the [Cloudstic CLI documentation](https://docs.cloudstic.com) and the [open-source GitHub repository](https://github.com/cloudstic/cli).

## Why Profiles Matter

Without profiles, every backup command needs to carry operational details:

- store URI
- repository password or key setup
- cloud credentials
- source URI
- OAuth token file paths
- tags and source-specific flags

That is manageable for one experiment. It is not manageable for a backup routine.

Profiles turn this:

```bash
cloudstic backup \
  -store s3:my-backups/cloudstic \
  -s3-region eu-west-1 \
  -password "$CLOUDSTIC_PASSWORD" \
  -source local:~/Documents \
  -tag laptop
```

into this:

```bash
cloudstic backup -profile documents
```

That one change sounds small, but it has real benefits:

- automation gets simpler
- secret handling becomes centralized
- adding a second source does not duplicate configuration everywhere
- moving between machines becomes much less error-prone

## The Practical Goal

Let us build a setup that looks like this:

- `home-s3` store for encrypted backups in S3-compatible storage
- `documents` profile for `~/Documents`
- `pictures` profile for `~/Pictures`
- `google-work` auth entry for a Google Drive source
- `work-drive` profile for a Google Drive backup

By the end, daily backup commands will look like this:

```bash
cloudstic backup -profile documents
cloudstic backup -profile pictures
cloudstic backup -profile work-drive
```

Or all at once:

```bash
cloudstic backup -all-profiles
```

## Prerequisites

- [Cloudstic CLI](https://github.com/cloudstic/cli) installed
- A backup destination such as a local disk, S3-compatible bucket, Backblaze B2 bucket, or SFTP server
- A repository password you are willing to store via secret references
- For cloud sources, an OAuth token file or provider credentials already set up

## Step 1: Create a Named Store

First define the destination once.

Here is an S3 example:

```bash
cloudstic store new \
  -name home-s3 \
  -uri s3:my-backup-bucket/cloudstic \
  -s3-region eu-west-1
```

This creates or updates a named store entry in `profiles.yaml`.

You only need to define the backend once. Every profile can then point at `home-s3` with `-store-ref home-s3`.

If you prefer local storage for testing, the same pattern works:

```bash
cloudstic store new \
  -name local-disk \
  -uri local:/Volumes/BackupDrive/cloudstic
```

## Step 2: Initialize the Repository Once

Defining a store is configuration. Initializing it creates the repository.

```bash
cloudstic store init home-s3
```

Cloudstic will set up the encrypted repository and create key slots.

If you want a recovery phrase as well, initialize with that in mind during your first setup flow. The important operational point is this: **repository encryption and runtime credential storage are separate things**.

- repository encryption protects the backup data
- profile secret references tell the CLI how to obtain passwords and API keys at runtime

That distinction matters when you design your setup.

## Step 3: Create Local Backup Profiles

Now define what should be backed up.

For `~/Documents`:

```bash
cloudstic profile new \
  -name documents \
  -source local:~/Documents \
  -store-ref home-s3
```

For `~/Pictures`:

```bash
cloudstic profile new \
  -name pictures \
  -source local:~/Pictures \
  -store-ref home-s3
```

At this point, your daily backup flow is already much better:

```bash
cloudstic backup -profile documents
cloudstic backup -profile pictures
```

No repeated store URI. No repeated region. No repeated credential flags.

## Step 4: Add a Cloud Auth Entry

Profiles can also reference reusable OAuth settings for cloud sources.

Let us create a Google auth entry:

```bash
cloudstic auth new \
  -name google-work \
  -provider google \
  -google-token-file ~/.config/cloudstic/tokens/google-work.json
```

Then log in once:

```bash
cloudstic auth login -name google-work
```

Now attach that auth entry to a backup profile:

```bash
cloudstic profile new \
  -name work-drive \
  -source "gdrive-changes:/" \
  -store-ref home-s3 \
  -auth-ref google-work
```

That gives you a native Google Drive backup command that is still short and repeatable:

```bash
cloudstic backup -profile work-drive
```

## Step 5: Store Secrets the Right Way

This is the part many backup guides skip.

Do not put secrets directly in `profiles.yaml`.

Instead, store **secret references** there.

Cloudstic supports these practical patterns today:

- `env://VAR_NAME`
- `keychain://service/account` on macOS

### Good: Environment Variables

Environment variables are the simplest portable option, especially for:

- headless servers
- CI jobs
- containers
- Linux systems without a desktop keyring

Example references inside `profiles.yaml`:

```yaml
stores:
  home-s3:
    uri: s3:my-backup-bucket/cloudstic
    s3_region: eu-west-1
    s3_access_key_secret: env://AWS_ACCESS_KEY_ID
    s3_secret_key_secret: env://AWS_SECRET_ACCESS_KEY
    password_secret: env://CLOUDSTIC_PASSWORD
```

Then export the real values in your shell, systemd unit, launchd plist, or CI secret store:

```bash
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export CLOUDSTIC_PASSWORD="use-a-real-password-manager"
```

This keeps the config file shareable while the actual secret values stay outside it.

### Better on macOS: Keychain References

If you are on macOS and running backups from your own machine, Keychain is more convenient than re-exporting secrets in every shell.

Example store configuration:

```yaml
stores:
  home-s3:
    uri: s3:my-backup-bucket/cloudstic
    s3_region: eu-west-1
    s3_access_key_secret: env://AWS_ACCESS_KEY_ID
    s3_secret_key_secret: keychain://cloudstic/home-s3/s3-secret-key
    password_secret: keychain://cloudstic/home-s3/repo-password
```

In that model:

- the access key ID comes from the environment
- the S3 secret key comes from macOS Keychain
- the repository password comes from macOS Keychain

This is a very practical split. Non-sensitive identifiers can live in env vars, while higher-value secrets stay in the system keychain.

### What Not to Do

Avoid these habits:

- hardcoding repository passwords directly in shell scripts
- committing `profiles.yaml` with raw secret values
- pasting cloud credentials into terminal history
- using one giant command copied between machines with slight edits

Backups should reduce operational risk, not create a second secret-management problem.

## What `profiles.yaml` Can Look Like

Here is a realistic example with one store, one OAuth entry, and three profiles:

```yaml
stores:
  home-s3:
    uri: s3:my-backup-bucket/cloudstic
    s3_region: eu-west-1
    s3_access_key_secret: env://AWS_ACCESS_KEY_ID
    s3_secret_key_secret: keychain://cloudstic/home-s3/s3-secret-key
    password_secret: keychain://cloudstic/home-s3/repo-password

auth:
  google-work:
    provider: google
    google_token_file: ~/.config/cloudstic/tokens/google-work.json

profiles:
  documents:
    source: local:~/Documents
    store_ref: home-s3

  pictures:
    source: local:~/Pictures
    store_ref: home-s3

  work-drive:
    source: gdrive-changes:/
    store_ref: home-s3
    auth_ref: google-work
```

This is the core idea of practical backups:

- stores define destinations and secret references
- auth entries define reusable cloud login settings
- profiles define actual backup jobs

## Step 6: Inspect and Verify Before You Trust It

Before putting a backup workflow on autopilot, inspect it.

List what is configured:

```bash
cloudstic profile list
```

Show one profile with resolved references:

```bash
cloudstic profile show documents
```

Verify that a store can resolve credentials and unlock correctly:

```bash
cloudstic store verify home-s3
```

That last command is especially useful after rotating credentials or moving to a new machine.

## Step 7: Run the Actual Backup Routine

Once the store and profiles exist, your real-world workflow becomes boring in the best way.

Back up one profile:

```bash
cloudstic backup -profile documents
```

Back up everything enabled in `profiles.yaml`:

```bash
cloudstic backup -all-profiles
```

Check snapshots afterward:

```bash
cloudstic list
```

Then restore the snapshot you actually want:

```bash
cloudstic restore <snapshot-hash> -output ./restore.zip
```

That detail matters when you use multiple profiles in one repository. `latest`
means the latest snapshot overall, not necessarily the latest snapshot for your
`documents` profile.

That is what a practical setup should feel like. The complexity lives in configuration once, not in every daily command.

## A Good Pattern for Automation

After moving to profiles, automation becomes much safer.

For example, a scheduled job only needs to provide the secrets and invoke one stable command:

```bash
export AWS_ACCESS_KEY_ID=AKIA...
export AWS_SECRET_ACCESS_KEY=...
export CLOUDSTIC_PASSWORD="..."

cloudstic backup -all-profiles
```

No giant inline flag list. No repeated source declarations. No chance that one machine forgot `-s3-region` or pointed at the wrong bucket path.

On macOS, if you use Keychain-backed secret refs, the scheduled command can be even shorter because fewer secrets need to be exported in the job itself.

## The Main Idea

The best backup setup is not the one with the most features.

It is the one you will actually keep running six months from now.

Profiles help because they make backups operationally boring:

- short commands
- reusable destinations
- reusable auth configuration
- safer secret handling
- simpler automation

If you are still backing things up with hand-written one-off commands, this is the upgrade that usually makes the workflow stick.

For the complete command reference, see the [Cloudstic CLI documentation](https://docs.cloudstic.com).

If you want to inspect the code or try the project yourself, the CLI is open source on [GitHub](https://github.com/cloudstic/cli).
