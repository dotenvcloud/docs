---
title: Migrating from .env Files
slug: migrating-from-dotenv-files
order: 5
description: Import your existing .env files into DotEnv with dotenv push, then pull them everywhere instead of copying files around.
tags: [use-cases, migration, dotenv-files, push, pull, import, secrets]
---

# Migrating from .env Files

If you already have `.env` files scattered across laptops, servers, and a password manager,
migrating to DotEnv is a one-time import: push each existing file up to the right level, verify
it, then switch everyone over to `dotenv pull`. After that the files in your repo and on your
machines are disposable.

## Before you start

- Install and authenticate the CLI:

  ```bash
  curl -sSL https://dotenv.cloud/install.sh | bash
  dotenv login
  ```

- Make sure the project exists in DotEnv (create it with `dotenv project create` — see the
  [command reference](/documentation/cli/commands)).
- Gather the `.env` files you want to import and note which stage each belongs to.

## Import a single file

`dotenv push` uploads a local file as the encrypted secrets for one level. Push each `.env` to
the path that matches its stage:

```bash
# Your existing development file
dotenv push myapp/development/local .env

# Your existing production file
dotenv push myapp/production/api .env.production
```

If the level already has secrets, `push` asks before overwriting unless you pass `--force`.

## Put shared values at the project level

If several of your files repeat the same values, lift those into the project level so they're
defined once and inherited everywhere:

```bash
# Values common to every stage
dotenv push myapp .env.defaults
```

Then trim those duplicated keys from the stage-specific files before pushing them, so each value
lives in exactly one place. See [multiple environments](/documentation/use-cases/multi-environment)
for how inheritance works.

## Import several files at once

For a project where you have separate project, target, and environment files, push them together
and the CLI will prompt you to place each one:

```bash
dotenv push myapp --project=.env.project --target=.env.target --env=.env.env
```

## Verify the import

Pull the level back down and compare it to your original file before you delete anything:

```bash
dotenv pull myapp/production/api --output /tmp/check.env --quiet
diff <(sort .env.production) <(sort /tmp/check.env)
rm /tmp/check.env
```

A clean diff (or only ordering differences) means the import is faithful.

## Switch everyone to pull

Once the values are in DotEnv and verified:

1. Replace local files by pulling instead:

   ```bash
   dotenv pull myapp/development/local --output .env
   ```

2. Update your CI/CD to pull with a read-only key — see
   [CI/CD pipelines](/documentation/use-cases/ci-cd).
3. Make sure `.env` files are gitignored so old copies can't drift back in:

   ```gitignore
   .env
   .env.*
   !.env.example
   ```

From here on, the source of truth is DotEnv. The local files are just a cache you can regenerate
at any time with `dotenv pull`.

## Where to go next

- [Pulling and pushing](/documentation/cli/pulling-and-pushing) — push modes and options.
- [Team collaboration](/documentation/use-cases/team-collaboration) — get the whole team pulling.
- [Multiple environments](/documentation/use-cases/multi-environment) — organize the imported
  values.
- [CI/CD pipelines](/documentation/use-cases/ci-cd) — wire pulls into automation.
