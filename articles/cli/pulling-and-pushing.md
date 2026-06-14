---
title: Pulling and Pushing
slug: pulling-and-pushing
order: 5
description: Use dotenv pull and push in depth — paths, output, formats, interpolation, and hierarchy merging.
tags: [cli, pull, push, secrets, env, formats, hierarchy]
---

# Pulling and Pushing

`dotenv pull` reads secrets from the cloud; `dotenv push` writes a local file back up. Both
operate on the `project/target/environment` hierarchy.

## Paths

A path identifies how deep into the hierarchy you are acting:

```
myapp                     # project level
myapp/production          # target level
myapp/production/api      # environment level
```

## Pulling

By default `pull` writes the result to stdout in `.env` format:

```bash
dotenv pull myapp/production/api
```

### Writing to a file

Use `--output` (`-o`) to write to a file. The CLI creates parent directories as needed, and if
the target file already exists it offers to back it up to `<file>.backup` first:

```bash
dotenv pull myapp/production/api --output .env
```

### Hierarchy merging

`pull` merges every level in the path by default (`--merge`, on by default). The merge order is
**project → target → environment**, with deeper levels overriding shallower ones. So a value
set on the project applies everywhere unless a target or environment redefines it.

To read only the deepest level you named — ignoring inherited parent values — use
`--level-only`:

```bash
# Only the secrets defined directly on the api environment
dotenv pull myapp/production/api --level-only
```

`--level-only` implies `--merge=false`.

### Output formats

`--format` (`-f`) selects the output format. Valid values:

| Format | Output |
|--------|--------|
| `env` (default) | `KEY=value` lines (`.env` format). |
| `json` | A JSON object of key/value pairs. |
| `yaml` (or `yml`) | YAML mapping. |
| `shell` | `export KEY="value"` statements (shell-escaped). |
| `dockerfile` | `ENV KEY="value"` lines for a Dockerfile. |

```bash
dotenv pull myapp/production/api --format=json --output secrets.json
```

### Variable interpolation

Use `--resolve` (`-r`) to expand `${VAR}` references between secrets before output:

```bash
dotenv pull myapp/production/api --resolve
```

If a reference cannot be resolved, the CLI warns and leaves the raw value in place.

### Decryption

Decryption is on by default (`--decrypt=true`). For client-managed projects the CLI resolves
the encryption key (see [Client-side encryption](./client-side-encryption.md)). To retrieve the
raw encrypted blobs without decrypting, pass `--decrypt=false`:

```bash
dotenv pull myapp/production/api --decrypt=false
```

### Quiet mode

`--quiet` (`-q`) suppresses all non-error output so the command communicates only through its
exit code — useful for scripting checks.

## Pushing

`push` uploads a local file as the encrypted secret blob for one level. The whole file is
stored as a single encrypted blob per level (the inverse of pull).

### Single file to one level

```bash
# Push .env to the api environment
dotenv push myapp/production/api .env

# Push defaults to the project level (inherited by all targets/environments)
dotenv push myapp .env.defaults
```

If the target level already has secrets, `push` asks for confirmation before overwriting unless
you pass `--force` (`-f`).

### Multiple files across levels

In multi-file mode, name only the project and pass a file per level. You will be prompted to
select the target/environment for the target- and environment-level files:

```bash
dotenv push myapp --project=.env.project --target=.env.target --env=.env.env
```

### Encryption on push

Encryption is on by default (`--encrypt=true`). For client-managed projects the CLI derives a
key proof and the server verifies it before accepting the write, so you cannot accidentally
orphan a level with the wrong key. Pushing plaintext to a client-managed project
(`--encrypt=false`) requires an explicit confirmation.

### Skipping the backup version

Every push records a backup version by default so you can roll back. To skip recording a backup
version for a particular push:

```bash
dotenv push myapp/production/api .env --no-backup
```

## Exporting

`dotenv export` is `pull` tuned for producing output in a given format (decryption on,
interpolation off):

```bash
dotenv export myapp/production/api --format=shell > exports.sh
dotenv export myapp --output=secrets.json --format=json
```

## Common patterns

```bash
# Local development: refresh your .env
dotenv pull myapp/development/local --output .env

# CI: fetch production secrets non-interactively
DOTENV_API_KEY=xxxxx dotenv pull myapp/production/api --output .env --quiet

# Build a container env file
dotenv export myapp/production/api --format=dockerfile --output Dockerfile.env
```
