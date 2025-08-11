# CRITICAL DEVELOPMENT REMINDERS

<critical>
- NEVER make assumptions about how ANY code works. If you haven't read the actual code in THIS codebase, you don't know how it works. Period.
</critical>

## Cross-Repository Work
- ALL secrets use AES-256-GCM encryption (32-byte keys, 12-byte IV, base64(IV + ciphertext + tag))
- Maintain API compatibility across CLI/SDKs when changing apps/web endpoints
- Test cross-repo changes with integration tests before commits
- Version synchronization critical for releases

## Repository-Specific Reminders
- **apps/web**: See `apps/web/docs/reminder.md` - TALL stack, Action Pattern
- **apps/cli**: See `apps/cli/docs/reminder.md` - Go/Cobra, client encryption
- **packages/sdk-php**: See `packages/sdk-php/docs/reminder.md` - PSR, API client
- **packages/sdk-js**: See `packages/sdk-js/docs/reminder.md` - TS, dual environment
- **packages/sdk-go**: See `packages/sdk-go/docs/reminder.md` - Shared with CLI

## Architecture Reference
- Individual `CLAUDE.md` files contain repo-specific context