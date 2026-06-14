---
title: Audit Logging
slug: audit-logging
order: 3
description: What the DotEnv activity and audit log records, how to read it, and how it supports compliance and incident review.
tags: [security, audit, activity-log, compliance, logging, accountability]
---

# Audit Logging

DotEnv keeps an **activity log** (audit log) of meaningful actions in your organization so you can
answer the questions that matter during a review or incident: *who did what, to which resource,
and when.*

## What is tracked

The log records security- and configuration-relevant events across the organization, including:

- **Secrets:** writes, restores of older versions, and history/purge operations. Secret *values*
  are never recorded — only that an action occurred, on which secret, by whom.
- **Encryption keys:** key creation, rotation, and changes to active state. Key material and
  proofs are never logged; entries capture the version, active flag, rotation timestamp, and a
  non-sensitive key hint.
- **Access control:** member invitations and removals, role changes, and team membership.
- **API tokens:** creation, update, and deletion of tokens (the token secret itself is never
  logged).
- **Projects, targets, and environments:** create, update, and delete operations.
- **Account & organization security:** two-factor changes, session/device activity, and
  organization settings such as 2FA enforcement.

Each entry captures the **actor**, the **action**, the **affected resource**, and a **timestamp**.
Sensitive material — secret values, encryption keys, passphrases, key-check proofs, and token
secrets — is **deliberately excluded** from the log.

## Where to view it

The activity log lives in the web dashboard under your organization's activity view. From there
you can review recent events and trace the history of a resource. See
[Notifications & Activity](/documentation/web-app/notifications-and-activity) for the full
walkthrough of reading and filtering the log.

## Read-only auditing

For compliance and review work, assign the **Auditor** role. It grants read access to projects,
environments, secret *metadata and version history*, members, and the audit log — but **no
decrypt permission**, so an auditor can verify activity without ever seeing secret values. See
[Access Control & Best Practices](/documentation/security-compliance/best-practices) for the full
role matrix.

## Using the log for compliance and incidents

- **Accountability:** every privileged change is attributable to an actor, supporting
  least-privilege reviews and access recertification.
- **Incident response:** if a key or token is suspected compromised, the log shows when it was
  used, rotated, or revoked, helping you scope the blast radius.
- **Change review:** rotation, role changes, and token lifecycle events give you a verifiable
  history of your security posture over time.

## Related

- [Notifications & Activity (web)](/documentation/web-app/notifications-and-activity)
- [Best Practices](/documentation/security-compliance/best-practices)
- [Key Management](/documentation/security-compliance/key-management)
