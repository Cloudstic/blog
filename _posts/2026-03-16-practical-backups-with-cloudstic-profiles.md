---
layout: post
title: "Practical Backups with Cloudstic Profiles"
date: 2026-03-16
author: loichrn
categories: backup cli
description: >-
  Learn how to turn Cloudstic CLI into a practical daily backup workflow using
  profiles, named stores, reusable auth entries, and secret references instead
  of long one-off commands.
tags: [backup, cli, profiles, secrets, s3, google-drive, automation]
---

The first backup command is easy.

The problem starts when you want to keep running it.

## Prerequisites

Before creating stores, auth entries, or profiles, make sure you have:

- the Cloudstic CLI installed by following the [installation guide](https://docs.cloudstic.com/installation)
- a backup destination already chosen, such as local storage or an S3-compatible bucket
- the credentials for that destination available through environment variables or your system secret manager
- a writable Cloudstic config location on the machine where you are defining the profiles

At the beginning, many people do something like this:

```bash
cloudstic backup \
  -store s3:my-backups/cloudstic \
  -password "correct horse battery staple" \
  -source local:~/Documents
```

This works.

It is also how you end up with secrets in shell history, slightly different commands on each machine, and an automation story that becomes fragile very quickly.

This is why I added profiles.

## Why Profiles Matter

For a backup workflow, the command itself should be short.

The complexity should live in configuration that you define once.

Profiles let you separate three things:

- where backups are stored
- how a cloud source authenticates
- what each backup job actually backs up

This is not about adding abstraction for the sake of abstraction.

It is about making the workflow repeatable.

## The Goal

Let us say you want:

- one S3-backed repository
- one profile for `~/Documents`
- one profile for `~/Pictures`
- one Google Drive profile for work files

The end result should look like this:

```bash
cloudstic backup -profile documents
cloudstic backup -profile pictures
cloudstic backup -profile work-drive
```

Or, if you want everything:

```bash
cloudstic backup -all-profiles
```

That is the practical value.

## Step 1: Define the Store Once

First define the destination.

For example, an S3-backed store:

```bash
cloudstic store new \
  -name home-s3 \
  -uri s3:my-backup-bucket/cloudstic \
  -s3-region eu-west-1
```

At this point, the destination has a name.

You no longer need to repeat the bucket path and region in every command.

## Step 2: Initialize the Repository

Configuration alone does not create the repository.

You still need to initialize it once:

```bash
cloudstic store init home-s3
```

This creates the encrypted repository and its key slots.

The important distinction is that repository encryption and runtime secret resolution are two different concerns.

The repository protects the backup data.

Profiles tell the CLI how to obtain the credentials needed to operate.

## Step 3: Create Local Profiles

Now define what should be backed up.

For documents:

```bash
cloudstic profile new \
  -name documents \
  -source local:~/Documents \
  -store-ref home-s3
```

For pictures:

```bash
cloudstic profile new \
  -name pictures \
  -source local:~/Pictures \
  -store-ref home-s3
```

At this stage, daily commands already become simpler:

```bash
cloudstic backup -profile documents
cloudstic backup -profile pictures
```

## Step 4: Reuse Cloud Authentication

The same idea also applies to cloud sources.

For example, define a Google auth entry once:

```bash
cloudstic auth new \
  -name google-work \
  -provider google \
  -google-token-file ~/.config/cloudstic/tokens/google-work.json
```

Then authenticate once:

```bash
cloudstic auth login -name google-work
```

Now create a profile that uses it:

```bash
cloudstic profile new \
  -name work-drive \
  -source "gdrive-changes:/" \
  -store-ref home-s3 \
  -auth-ref google-work
```

The resulting command stays short:

```bash
cloudstic backup -profile work-drive
```

## Step 5: Do Not Put Raw Secrets in the Config

This is the part that matters most operationally.

Do not store raw secrets directly in `profiles.yaml`.

Store references instead.

The two practical patterns are:

- `env://VAR_NAME`
- `keychain://service/account` on macOS

For example:

```yaml
stores:
  home-s3:
    uri: s3:my-backup-bucket/cloudstic
    s3_region: eu-west-1
    s3_access_key_secret: env://AWS_ACCESS_KEY_ID
    s3_secret_key_secret: keychain://cloudstic/home-s3/s3-secret-key
    password_secret: keychain://cloudstic/home-s3/repo-password
```

This keeps the configuration file shareable while the actual secret values stay outside it.

On a server or in CI, environment variables are usually the easiest option.

On a personal macOS machine, Keychain is often more convenient.

## What the File Looks Like

With one store, one auth entry, and three profiles, the configuration stays readable:

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

This is the main idea:

- stores define destinations
- auth entries define reusable cloud login configuration
- profiles define backup jobs

## Step 6: Verify Before Automating

Before trusting the setup, inspect it.

List profiles:

```bash
cloudstic profile list
```

Show one profile:

```bash
cloudstic profile show documents
```

Verify the store:

```bash
cloudstic store verify home-s3
```

This is especially useful after rotating credentials or moving to a new machine.

## Step 7: Run the Routine

Once the setup exists, the real workflow becomes boring, which is exactly what I want from a backup system.

Run one profile:

```bash
cloudstic backup -profile documents
```

Run everything:

```bash
cloudstic backup -all-profiles
```

Inspect snapshots:

```bash
cloudstic list
```

Restore what you need:

```bash
cloudstic restore <snapshot-hash> -output ./restore.zip
```

## Conclusion

For me, profiles are not a cosmetic CLI feature.

They are the point where backup commands stop being one-off experiments and become an actual workflow.

The benefit is simple:

- shorter commands
- cleaner automation
- safer secret handling
- less duplication
- less room for operational mistakes

If the command you run every day still contains every flag, every secret, and every path inline, the setup is not finished yet.

To get started, see the [installation guide](https://docs.cloudstic.com/installation). For the complete command reference, see the [Cloudstic CLI documentation](https://docs.cloudstic.com). The CLI is also open source on [GitHub](https://github.com/cloudstic/cli).
