---
title: Billing
slug: billing
order: 11
description: View plans, subscriptions, usage, and invoices, and manage subscriptions, payment methods, charges, and coupons via the billing API.
tags: [api, billing, subscriptions, invoices, payment-methods]
---

# Billing

The billing API exposes plans, the organization's subscription, usage, invoices, payment methods,
one-time charges, and coupons. Read endpoints require `billing:read`; mutations require
`billing:write`. Mutating endpoints also have per-action rate limits.

There are two surfaces:

- A **flat surface** under `/api/v1/organizations/{organization}/...` (subscription, usage,
  invoices, payment-methods).
- A **structured surface** under `/api/v1/organizations/{organization}/billing/...` with the full
  subscription / payment-method / invoice / charge / coupon lifecycle.

Plans are a global catalog and are org-agnostic.

## Plans (global)

| Method | Path | Scope |
| --- | --- | --- |
| `GET` | `/api/v1/plans` | — (authenticated) |

```bash
curl https://api.dotenv.cloud/api/v1/plans \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

## Flat surface

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `GET` | `/api/v1/organizations/{organization}/subscription` | `billing:read` | Current subscription summary (`404` if no plan). |
| `GET` | `/api/v1/organizations/{organization}/usage` | `billing:read` | Usage statistics. |
| `GET` | `/api/v1/organizations/{organization}/invoices` | `billing:read` | Recent billing transactions (latest 50). |
| `POST` | `/api/v1/organizations/{organization}/payment-methods` | `billing:write` | Add a payment method. |
| `POST` | `/api/v1/organizations/{organization}/subscription` | `billing:write` | Create a subscription. |
| `PATCH` | `/api/v1/organizations/{organization}/subscription` | `billing:write` | Update a subscription. |
| `DELETE` | `/api/v1/organizations/{organization}/subscription` | `billing:write` | Cancel a subscription. |

```bash
# Current subscription
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/subscription \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

# Usage
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/usage \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"

# Create a subscription
curl -X POST https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/subscription \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{ "plan_id": "...", "billing_interval": "month" }'
```

## Structured surface (`/billing`)

### Subscriptions

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `GET` | `/api/v1/organizations/{organization}/billing/subscription` | `billing:read` | Show subscription. |
| `POST` | `/api/v1/organizations/{organization}/billing/subscription` | `billing:write` | Create subscription. |
| `PATCH` | `/api/v1/organizations/{organization}/billing/subscription` | `billing:write` | Update subscription. |
| `DELETE` | `/api/v1/organizations/{organization}/billing/subscription` | `billing:write` | Cancel subscription. |
| `POST` | `/api/v1/organizations/{organization}/billing/subscription/resume` | `billing:write` | Resume a canceled subscription. |

```bash
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/billing/subscription \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

### Payment methods

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `GET` | `/api/v1/organizations/{organization}/billing/payment-methods` | `billing:read` | List payment methods. |
| `POST` | `/api/v1/organizations/{organization}/billing/payment-methods` | `billing:write` | Add a payment method. |
| `PATCH` | `/api/v1/organizations/{organization}/billing/payment-methods/{paymentMethodId}/default` | `billing:write` | Set default payment method. |
| `DELETE` | `/api/v1/organizations/{organization}/billing/payment-methods/{paymentMethodId}` | `billing:write` | Remove a payment method. |

```bash
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/billing/payment-methods \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

### Charges & refunds

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `POST` | `/api/v1/organizations/{organization}/billing/charges` | `billing:write` | Create a one-time charge. |
| `POST` | `/api/v1/organizations/{organization}/billing/charges/{chargeId}/refund` | `billing:write` | Refund a charge. |

```bash
curl -X POST https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/billing/charges \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{ "amount": 5000, "currency": "usd", "description": "One-time setup" }'
```

### Invoices

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `GET` | `/api/v1/organizations/{organization}/billing/invoices` | `billing:read` | List invoices. |
| `GET` | `/api/v1/organizations/{organization}/billing/invoices/{invoiceId}` | `billing:read` | Get an invoice. |
| `GET` | `/api/v1/organizations/{organization}/billing/invoices/{invoiceId}/download` | `billing:read` | Download an invoice. |

```bash
curl https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/billing/invoices \
  -H "Authorization: Bearer <token>" -H "Accept: application/json"
```

### Coupons

| Method | Path | Scope | Description |
| --- | --- | --- | --- |
| `POST` | `/api/v1/organizations/{organization}/billing/coupons` | `billing:write` | Apply a coupon. |
| `DELETE` | `/api/v1/organizations/{organization}/billing/coupons` | `billing:write` | Remove a coupon. |

```bash
curl -X POST https://api.dotenv.cloud/api/v1/organizations/01ARZ3NDEKTSV4RRFFQ69G5FAV/billing/coupons \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{ "coupon": "LAUNCH20" }'
```
