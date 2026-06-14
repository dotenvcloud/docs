---
title: Account & Security
slug: account-and-security
order: 9
description: Manage your profile, password, two-factor authentication methods, trusted devices and sessions, and review third-party OAuth authorizations.
tags: [web-app, account, security, 2fa, totp, sessions, trusted-devices, oauth]
---

# Account & Security

Your personal account is managed at **Account Settings** (`/account/settings`), separate from any
organization. It has four sections:

| Section | URL | What it covers |
| --- | --- | --- |
| **Profile** | `/account/settings/profile` | Your name and avatar. |
| **Security** | `/account/settings/security` | Password, 2FA, sessions, trusted devices. |
| **Preferences** | `/account/settings/preferences` | Personal UI preferences. |
| **Notifications** | `/account/settings/notifications` | Notification and email preferences. |

Account settings stay accessible even when your organization **enforces 2FA**, so you can always
set up the methods you need to comply.

## Profile

In **Profile** you can update your **name** and manage your **avatar** (upload an image or choose
how your avatar is generated, and remove it). Save to apply the changes.

## Password

In **Security**, open the password form to change your password. You confirm your current password
and set a new one; the page shows when your password was last changed.

## Two-factor authentication (2FA)

DotEnv supports multiple 2FA methods, and you can have more than one configured:

- **Authenticator app (TOTP)** — scan a QR code with an app like 1Password, Authy, or Google
  Authenticator, then confirm a code.
- **Email codes** — receive a one-time code by email.
- **SMS codes** — receive a one-time code by text message.

From the **Security** section you can:

- **Enable** any of the methods (authenticator, email, or SMS).
- See your **configured methods** and which is primary.
- **View and regenerate backup (recovery) codes** — one-time codes that let you sign in if you
  lose access to your other methods. Save them somewhere safe.
- **Disable** 2FA.

When you sign in with 2FA active, you'll be asked for a code at the two-factor challenge step, and
you can switch between your configured methods or request a resend.

> If your organization enforces 2FA, you must keep at least one method enabled to retain access
> after the grace period. See [Organizations](organizations).

## Trusted devices and sessions

The **Security** section also lets you manage where you're signed in:

- Review recent **login history**.
- **Revoke a device** you no longer trust or recognize, so it can no longer skip the 2FA
  challenge.

If you spot a device or session you don't recognize, revoke it and change your password.

## Preferences

The **Preferences** section holds personal UI choices (such as editor and display preferences)
that apply to your account across the dashboard.

## Third-party (OAuth) authorizations

DotEnv can authorize third-party applications and the CLI to act on your behalf through an OAuth
flow. When such an app requests access, you're taken to an **authorization page**
(`/oauth/authorize`) that shows what is requesting access and what it will be allowed to do. You
then **approve** or decline.

Only approve requests you initiated and recognize. If you authorize the CLI or an integration and
later want to revoke it, rotate the relevant [API key](api-keys) or credentials so the access no
longer works.
