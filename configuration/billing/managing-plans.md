# Managing your plan

> **Base URL:** `https://platform.ibl.ai`
> **Authentication:** `Authorization: Token YOUR_ACCESS_TOKEN`
> **Applies to:** Platform admins, owners, and end-users

---

## Table of Contents

1. [What Plan Am I On?](#1-what-plan-am-i-on)
   - [1.1 Credit Account (current plan + balance)](#11-credit-account-current-plan--balance)
2. [What Is My Credit Balance?](#2-what-is-my-credit-balance)
   - [2.1 Get Balance](#21-get-balance)
   - [2.2 Transaction History](#22-transaction-history)
3. [How Do I Upgrade?](#3-how-do-i-upgrade)
   - [3.1 Upgrade via the UI](#31-upgrade-via-the-ui)

---

## 1. What Plan Am I On?

Check your current plan depending 

| System | Endpoint | When to use |
|--------|----------|-------------|
| **Credit account** | `GET /api/billing/account/` | Platforms using the credit-based billing model |

---

### 1.1 Credit Account (current plan + balance)

#### GET `/api/billing/account/`

Returns the user's credit account including current plan, balance, and auto-recharge settings.

**Query parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `platform_key` | string | No | View a specific platform's account (admin only) |

**Response — `200 OK`:**

```json
{
  "has_credits": true,
  "account_id": "a1b2c3d4-...",
  "available_credits": "42.50",
  "current_plan": "growth",
  "previous_plan": "free_trial",
  "free_trial": false,
  "auto_recharge_enabled": true,
  "auto_recharge_threshold_usd": "5.00",
  "auto_recharge_amount_usd": "25.00",
  "auto_recharge_spending_limit_usd": "100.00",
  "auto_recharge_last_triggered_at": "2026-04-10T14:30:00Z",
  "can_use_auto_recharge": true,
  "has_payment_method": true,
  "pricing_table": { "...Stripe pricing table config..." },
  "platform_key": "my-platform"
}
```

**Key fields:**

| Field | Type | Description |
|-------|------|-------------|
| `current_plan` | string | Active plan identifier (e.g. `free_trial`, `growth`, or a Stripe product SKU) |
| `previous_plan` | string | Plan before the last purchase |
| `free_trial` | boolean | `true` if the user is on the free trial |
| `available_credits` | decimal | Current credit balance in USD |
| `has_credits` | boolean | `true` if credits > 0 or plan grants infinite credits |

**Example:**

```bash
curl -H "Authorization: Token YOUR_TOKEN" \
  https://app.ibl.ai/api/billing/account/
```

---
## 2. What Is My Credit Balance?

### 2.1 Get Balance

Use the same **Credit Account** endpoint described in [Section 1.1](#11-credit-account-current-plan--balance):

```
GET /api/billing/account/
```

The `available_credits` field is your current balance. `has_credits` tells you at a glance whether you can perform credit-consuming actions.

---

### 2.2 Transaction History

#### GET `/api/billing/transactions/`

Returns a paginated list of credit transactions (top-ups, deductions, refunds, etc.).

**Query parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `platform_key` | string | No | View platform account transactions (admin only) |
| `transaction_type` | string | No | Filter: `add`, `subtract`, `reserve`, `release`, `rollover`, `refund` |
| `from_date` | string | No | Start date (`YYYY-MM-DD`) |
| `to_date` | string | No | End date (`YYYY-MM-DD`) |
| `page` | integer | No | Page number |
| `page_size` | integer | No | Items per page (max 100, default 20) |

**Response — `200 OK`:**

```json
{
  "count": 150,
  "results": [
    {
      "id": "c3d4e5f6-...",
      "transaction_type": "subtract",
      "status": "completed",
      "credits_amount": "0.03",
      "credits_balance_after": "42.47",
      "description": "AI Agent session — Premium Tutor",
      "created_at": "2026-04-17T09:15:00Z"
    },
    {
      "id": "d4e5f6a7-...",
      "transaction_type": "add",
      "status": "completed",
      "payment_amount_usd": "25.00",
      "credits_amount": "25.00",
      "credits_balance_after": "42.50",
      "description": "Auto-recharge",
      "created_at": "2026-04-10T14:30:00Z"
    }
  ]
}
```

**Key fields:**

| Field | Type | Description |
|-------|------|-------------|
| `transaction_type` | string | `add` = top-up/purchase, `subtract` = usage, `refund` = credit returned |
| `payment_amount_usd` | decimal | Real payment amount (only present for `add` transactions) |
| `credits_amount` | decimal | Credits moved in the transaction |
| `credits_balance_after` | decimal | Running balance after the transaction |

**Example:**

```bash
curl -H "Authorization: Token YOUR_TOKEN" \
  "https://app.ibl.ai/api/billing/transactions/?from_date=2026-04-01&page_size=50"
```

---

## 3. How Do I Upgrade?

### 3.1 Upgrade via the UI

The recommended way to upgrade your plan is through the platform dashboard.

**Step 1 — Open Billing Settings**

Navigate to your platform's **Settings > Billing** page.

image.png

**Step 2 — Click Upgrade**

Select the plan that fits your needs from the pricing table.

**Step 3 — Complete Checkout**

You'll be redirected to Stripe's secure checkout page to enter payment details.


**Step 4 — Confirmation**

After payment, you'll be returned to the platform with your new plan active. Your credit balance and plan details update automatically.

---


## Quick Reference

| Question | Endpoint | Key response field |
|----------|----------|--------------------|
| What plan am I on? | `GET /api/billing/account/` | `current_plan`, `free_trial` |
| What's my credit balance? | `GET /api/billing/account/` | `available_credits`, `has_credits` |
| What's my transaction history? | `GET /api/billing/transactions/` | `results[]` |
