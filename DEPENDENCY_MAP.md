# DotEnv Dependency Map

This document maps how changes in the Web API affect the CLI and SDKs.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DotEnv Platform                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      apps/web (Laravel)                              │   │
│  │                    SOURCE OF TRUTH FOR API                           │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                  │   │
│  │  │ Controllers │  │   Actions   │  │  Services   │                  │   │
│  │  │   (API)     │  │ (Business)  │  │ (Encryption)│                  │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                  │   │
│  │         │                │                │                          │   │
│  │         └────────────────┼────────────────┘                          │   │
│  │                          ▼                                           │   │
│  │                 ┌─────────────────┐                                  │   │
│  │                 │  API Endpoints  │                                  │   │
│  │                 │   /api/v1/*     │                                  │   │
│  │                 └────────┬────────┘                                  │   │
│  └──────────────────────────┼──────────────────────────────────────────┘   │
│                             │                                               │
│                             ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                   tools/specs/openapi.yaml                            │  │
│  │                     API CONTRACT (68KB)                               │  │
│  │  - Endpoint definitions    - Request/Response schemas                │  │
│  │  - Authentication methods  - Error codes                             │  │
│  │  - Rate limiting rules     - Encryption format                       │  │
│  └──────────────────────────────┬───────────────────────────────────────┘  │
│                                 │                                           │
│           ┌─────────────────────┼─────────────────────┐                    │
│           ▼                     ▼                     ▼                    │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │  packages/      │   │  packages/      │   │  packages/      │          │
│  │  sdk-go         │   │  sdk-php        │   │  sdk-js         │          │
│  │  ─────────────  │   │  ─────────────  │   │  ─────────────  │          │
│  │  ✅ Complete    │   │  ⚠️ Skeleton    │   │  ⚠️ Skeleton    │          │
│  │  • client.go    │   │  • composer.json│   │  • package.json │          │
│  │  • secrets.go   │   │  • CLAUDE.md    │   │  • CLAUDE.md    │          │
│  │  • encryption.go│   │                 │   │                 │          │
│  │  • oauth.go     │   │                 │   │                 │          │
│  └────────┬────────┘   └─────────────────┘   └─────────────────┘          │
│           │                                                                │
│           │  go.mod replace directive                                      │
│           ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                        apps/cli (Go)                                 │  │
│  │  ─────────────────────────────────────────────────────────────────  │  │
│  │  Uses sdk-go via: replace github.com/dotenv/sdk-go => ../../packages/sdk-go │
│  │                                                                      │  │
│  │  Commands:                                                           │  │
│  │  • login.go    → OAuth endpoints                                    │  │
│  │  • pull.go     → Secrets endpoints + Encryption                     │  │
│  │  • push.go     → Secrets endpoints + Encryption                     │  │
│  │  • list.go     → Projects/Targets/Environments endpoints            │  │
│  │  • apikeys.go  → API Keys endpoints                                 │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## API Endpoint → SDK/CLI Dependency Matrix

### Authentication Endpoints

| API Endpoint | SDK-Go | CLI | SDK-PHP | SDK-JS | Impact Level |
|-------------|--------|-----|---------|--------|--------------|
| `POST /api/v1/oauth/token` | `oauth.go` | `login.go` | Planned | Planned | 🔴 Critical |
| `GET /api/v1/user` | `client.go` (UserService) | `account.go` | Planned | Planned | 🟡 Medium |

**Breaking Change Impact**: OAuth flow changes break ALL clients immediately.

### Organization & Project Hierarchy

| API Endpoint | SDK-Go | CLI | SDK-PHP | SDK-JS | Impact Level |
|-------------|--------|-----|---------|--------|--------------|
| `GET /api/v1/{org}/projects` | `projects.go` | `list.go` | Planned | Planned | 🟡 Medium |
| `GET /api/v1/{org}/{project}` | `projects.go` | `list.go` | Planned | Planned | 🟡 Medium |
| `GET /api/v1/{org}/{project}/targets` | `targets.go` | `list.go` | Planned | Planned | 🟡 Medium |
| `GET /api/v1/{org}/{project}/{target}/environments` | `environments.go` | `list.go` | Planned | Planned | 🟡 Medium |

**Breaking Change Impact**: URL format changes require SDK updates, but legacy format preserved.

### Secrets Endpoints (HIGHEST IMPACT)

| API Endpoint | SDK-Go | CLI | SDK-PHP | SDK-JS | Impact Level |
|-------------|--------|-----|---------|--------|--------------|
| `GET /api/v1/{org}/{project}/secrets` | `secrets.go:GetProjectSecrets()` | `pull.go` | Planned | Planned | 🔴 Critical |
| `GET /api/v1/{org}/{project}/{target}/{env}/secrets` | `secrets.go:GetProjectSecrets()` | `pull.go` | Planned | Planned | 🔴 Critical |
| `POST /api/v1/{org}/secrets/retrieve` | `secrets.go:RetrieveSecrets()` | `pull.go` | Planned | Planned | 🔴 Critical |
| `POST /api/v1/{org}/{project}/secrets/push` | `secrets.go:PushSecrets()` | `push.go` | Planned | Planned | 🔴 Critical |
| `PATCH /api/v1/{org}/{project}/secrets/{key}` | `secrets.go:Update()` | `push.go` | Planned | Planned | 🟡 Medium |
| `DELETE /api/v1/{org}/{project}/secrets/{key}` | `secrets.go:Delete()` | - | Planned | Planned | 🟡 Medium |

**Breaking Change Impact**: Secrets API is core functionality. Changes cascade to ALL consumers.

### Encryption Endpoints (CRITICAL)

| API Endpoint | SDK-Go | CLI | SDK-PHP | SDK-JS | Impact Level |
|-------------|--------|-----|---------|--------|--------------|
| `GET /api/v1/{org}/{project}/encryption-key` | `encryption.go:GetEncryptionKey()` | `pull.go` | Planned | Planned | 🔴 Critical |
| `POST /api/v1/{org}/{project}/secrets/rotate-client-keys` | `encryption.go:RotateClientKeys()` | - | Planned | Planned | 🔴 Critical |

**Breaking Change Impact**: Encryption format changes break decryption across ALL SDKs.

### API Key Management

| API Endpoint | SDK-Go | CLI | SDK-PHP | SDK-JS | Impact Level |
|-------------|--------|-----|---------|--------|--------------|
| `GET /api/v1/organizations/{org}/api-keys` | `apikeys.go` | `apikeys.go` | Planned | Planned | 🟡 Medium |
| `POST /api/v1/organizations/{org}/api-keys` | `apikeys.go` | `apikeys.go` | Planned | Planned | 🟡 Medium |
| `DELETE /api/v1/organizations/{org}/api-keys/{id}` | `apikeys.go` | `apikeys.go` | Planned | Planned | 🟡 Medium |
| `POST /api/v1/organizations/{org}/api-keys/{id}/rotate` | `apikeys.go` | `apikeys.go` | Planned | Planned | 🟡 Medium |

### Telemetry

| API Endpoint | SDK-Go | CLI | SDK-PHP | SDK-JS | Impact Level |
|-------------|--------|-----|---------|--------|--------------|
| `POST /api/v1/telemetry` | `telemetry.go` | All commands | - | - | 🟢 Low |

## Cross-Component Dependency Graph

```
                    ┌─────────────────────────────────┐
                    │         apps/web API            │
                    │    (Source of Truth)            │
                    └───────────────┬─────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    ▼                               ▼
        ┌───────────────────┐           ┌───────────────────┐
        │  Response Format  │           │  Encryption Algo  │
        │  (JSON:API spec)  │           │  (AES-256-GCM)    │
        └─────────┬─────────┘           └─────────┬─────────┘
                  │                               │
    ┌─────────────┼─────────────┐   ┌─────────────┼─────────────┐
    ▼             ▼             ▼   ▼             ▼             ▼
┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐   ┌───────┐
│sdk-go │   │sdk-php│   │sdk-js │   │sdk-go │   │sdk-php│   │sdk-js │
│parse  │   │parse  │   │parse  │   │encrypt│   │encrypt│   │encrypt│
└───┬───┘   └───────┘   └───────┘   └───┬───┘   └───────┘   └───────┘
    │                                   │
    └───────────────┬───────────────────┘
                    ▼
            ┌───────────────┐
            │   apps/cli    │
            │ (Consumer of  │
            │   sdk-go)     │
            └───────────────┘
```

## Encryption Compatibility Matrix

**Standard**: AES-256-GCM with 12-byte IV

| Component | Implementation | Format | Padding | Status |
|-----------|---------------|--------|---------|--------|
| apps/web | PHP OpenSSL | `base64(IV + ciphertext + tag)` | '0' pad to 32 bytes | ✅ Reference |
| sdk-go | Go crypto/aes | `base64(IV + ciphertext + tag)` | '0' pad to 32 bytes | ✅ Compatible |
| sdk-php | OpenSSL | `base64(IV + ciphertext + tag)` | '0' pad to 32 bytes | ⏳ Planned |
| sdk-js | WebCrypto/Node | `base64(IV + ciphertext + tag)` | '0' pad to 32 bytes | ⏳ Planned |
| apps/cli | Uses sdk-go | Same as sdk-go | Same as sdk-go | ✅ Compatible |

**Critical**: If web changes encryption format, ALL SDKs must update simultaneously.

## Change Impact Procedures

### 🔴 Critical Changes (Require Coordinated Release)

1. **Encryption format changes**
   - Update: apps/web → sdk-go → sdk-php → sdk-js → apps/cli
   - Test: Cross-SDK encryption/decryption compatibility
   - Rollout: Coordinated release with migration period

2. **OAuth flow changes**
   - Update: apps/web → sdk-go (oauth.go) → apps/cli (login.go)
   - Test: Full authentication flow across all clients
   - Rollout: Version bump with deprecation period

3. **Secrets API response format**
   - Update: apps/web → openapi.yaml → all SDKs
   - Test: Response parsing in all SDKs
   - Rollout: Support both formats during transition

### 🟡 Medium Impact Changes

1. **New endpoint additions**
   - Update: apps/web → openapi.yaml → SDKs (optional)
   - Test: New functionality in implementing SDKs
   - Rollout: SDKs can adopt at own pace

2. **Error code changes**
   - Update: apps/web → sdk-go (errors.go) → apps/cli (errors.go)
   - Test: Error handling across clients
   - Rollout: Backward compatible with fallback

### 🟢 Low Impact Changes

1. **Rate limit adjustments**
   - Update: apps/web only
   - Document: Update openapi.yaml
   - No SDK changes required

2. **Telemetry additions**
   - Update: apps/web → sdk-go (telemetry.go) → apps/cli
   - Optional adoption by other SDKs

## File-Level Dependency Map

### apps/web Changes → SDK-Go Updates Required

| Web File | SDK-Go Files Affected | CLI Files Affected |
|----------|----------------------|-------------------|
| `app/Http/Controllers/Api/SecretsController.php` | `secrets.go` | `pull.go`, `push.go` |
| `app/Services/EncryptionService.php` | `encryption.go` | All commands using encryption |
| `app/Http/Controllers/Api/OAuthController.php` | `oauth.go` | `login.go`, `auth.go` |
| `app/Http/Controllers/Api/ProjectsController.php` | `projects.go` | `list.go`, `explore.go` |
| `app/Http/Controllers/Api/ApiKeysController.php` | `apikeys.go` | `apikeys.go` |
| `routes/api.php` | URL patterns in all files | URL patterns in all files |

### SDK-Go Changes → CLI Updates Required

| SDK-Go File | CLI Files Affected | Reason |
|-------------|-------------------|--------|
| `client.go` | `root.go`, `helpers.go` | Client initialization |
| `secrets.go` | `pull.go`, `push.go` | Secret operations |
| `encryption.go` | `pull.go`, `push.go` | Encrypt/decrypt |
| `oauth.go` | `login.go`, `auth.go` | Authentication |
| `projects.go` | `list.go`, `explore.go` | Project listing |
| `errors.go` | `errors.go` | Error handling |

## Testing Requirements for Cross-Component Changes

### Integration Test Matrix

```bash
# Test 1: Encryption compatibility
apps/web encrypt → sdk-go decrypt  ✓
sdk-go encrypt → apps/web decrypt  ✓
sdk-go encrypt → sdk-php decrypt   (when implemented)
sdk-go encrypt → sdk-js decrypt    (when implemented)

# Test 2: API response parsing
apps/web response → sdk-go parse   ✓
apps/web response → sdk-php parse  (when implemented)
apps/web response → sdk-js parse   (when implemented)

# Test 3: CLI → API integration
apps/cli pull → apps/web API       ✓
apps/cli push → apps/web API       ✓
apps/cli login → apps/web OAuth    ✓
```

## Version Compatibility Matrix

| CLI Version | SDK-Go Version | Web API Version | Status |
|-------------|---------------|-----------------|--------|
| 1.0.x | 1.0.x | v1 | Current |
| 1.1.x | 1.0.x | v1 | Planned |
| 2.0.x | 2.0.x | v1, v2 | Future |

## Quick Reference: Change Checklist

When modifying **apps/web API**:
- [ ] Update `tools/specs/openapi.yaml`
- [ ] Check if `sdk-go` needs updates
- [ ] Check if `apps/cli` needs updates
- [ ] Run integration tests
- [ ] Update this dependency map if needed

When modifying **sdk-go**:
- [ ] Ensure compatibility with `apps/web` API
- [ ] Check if `apps/cli` needs updates
- [ ] Run `go test ./...`
- [ ] Test with actual API

When modifying **apps/cli**:
- [ ] Check if `sdk-go` has required functionality
- [ ] Run CLI tests
- [ ] Test against staging API

## MCP Server Availability

| Component | Serena MCP | Status |
|-----------|-----------|--------|
| apps/web | mcp__serena-web__ | ✅ Configured |
| apps/cli | mcp__serena-cli__ | ✅ Configured |
| packages/sdk-go | mcp__serena-sdk-go__ | ✅ Just configured |
| packages/sdk-php | - | ⏳ Awaiting config |
| packages/sdk-js | - | ⏳ Awaiting config |
