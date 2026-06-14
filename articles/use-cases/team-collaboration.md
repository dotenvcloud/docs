---
title: Team Collaboration
slug: team-collaboration
order: 4
description: Share secrets across a team with a single source of truth, organization roles, and a fast onboarding flow for new teammates.
tags: [use-cases, team, collaboration, organizations, roles, onboarding, secrets]
---

# Team Collaboration

Secrets shared over chat, copied between laptops, or buried in a `.env` someone emailed two years
ago are a liability. DotEnv gives a team one source of truth: secrets live in the organization,
everyone pulls the same values, and access is controlled centrally.

## One source of truth

Instead of each developer maintaining their own `.env`, the canonical values live in DotEnv under
`project/target/environment`. Every teammate runs the same command and gets the same result:

```bash
dotenv pull myapp/development/local --output .env
```

When someone updates a secret, everyone else picks it up on their next pull. No more "works on my
machine because my `.env` is different."

## Organizations and accounts

Secrets belong to an **organization**, and people join the organization to gain access. The CLI
holds one or more accounts and can switch the active organization:

```bash
# See which organizations you belong to
dotenv org list

# Switch the organization commands act on
dotenv org use acme-corp

# Check who you're acting as
dotenv status
```

A single person can hold several accounts at once — for example a personal account and a work
account — and switch between them with `dotenv account use`. See
[authentication](/documentation/cli/authentication) for managing accounts and organizations.

## Roles and access

Access is granted per organization, so you control who can read and who can change secrets:

- Grant **read access** to developers who only need to pull values.
- Reserve **write access** for the people responsible for managing each environment.
- Use **read-only API keys** for anything automated — see
  [CI/CD pipelines](/documentation/use-cases/ci-cd).

Because the hierarchy separates stages, you can keep production credentials limited to a smaller
group while everyone shares development values.

## Onboarding a new teammate

A new hire gets productive in three steps — and never has to be handed a live secret:

1. **Add them to the organization** so they have access.
2. **They install and log in:**

   ```bash
   curl -sSL https://dotenv.cloud/install.sh | bash
   dotenv login
   ```

3. **They pull and start working:**

   ```bash
   dotenv pull myapp/development/local --output .env
   ```

That's the whole setup. No secret ever travels over chat or email — it comes straight from DotEnv
to their machine, decrypted locally.

## Keep .env private

Remind every teammate that a pulled `.env` holds real values and must stay out of git. Commit a
`.gitignore` rule and a placeholder `.env.example` so the rule is in place before anyone clones:

```gitignore
.env
.env.*
!.env.example
```

## Where to go next

- [Local development](/documentation/use-cases/local-development) — the per-developer flow.
- [Multiple environments](/documentation/use-cases/multi-environment) — separate prod, staging,
  and dev access.
- [Migrating from .env files](/documentation/use-cases/migrating-from-dotenv-files) — import a
  team's existing files.
- [Authentication](/documentation/cli/authentication) — accounts, organizations, and API keys.
