# SydxAI WhatsApp CRM (wacrm) — Architecture & Serverless Deployment Guide

> **Official internal documentation for `sai8341/wacrm`.**  
> Licensed under MIT. Own, brand, customize, and deploy.

---

## 1. Executive Summary & Technology Stack

SydxAI WhatsApp CRM is a modern, full-featured WhatsApp Business CRM and automation platform designed to operate with zero legacy bloat, no ORMs, and no separate backend server.

| Component | Technology | Purpose |
| :--- | :--- | :--- |
| **Framework** | **Next.js 16 (App Router)** | React 19, Server Components for high performance, Client Components for real-time reactivity. |
| **UI & Styling** | **Tailwind CSS v4 + Base UI (shadcn)** | Zero-runtime CSS, dark-mode native, accessible design system. |
| **Database & Auth** | **Supabase (PostgreSQL 17)** | Row-Level Security (RLS) data isolation, Auth (HttpOnly SSR cookies), Vector embeddings (`pgvector`), Storage. |
| **Messaging Gateway** | **Official Meta WhatsApp Cloud API** | Direct Meta Graph API integration (no third-party unapproved scrapers/gateways like Baileys or Evolution). |
| **Cryptography** | **Node.js `node:crypto`** | AES-256-GCM token encryption at rest; HMAC-SHA256 inbound webhook signature verification. |
| **MCP Integration** | **Model Context Protocol** | Native MCP server in `mcp-server/` allowing AI agents (Claude, Cursor, Antigravity) to query and drive CRM data. |

---

## 2. Multi-Tenancy Architecture & Data Isolation

The platform utilizes a **Single-Database, Account-Scoped Multi-Tenant Architecture**:

```
[ auth.users ] (Login credentials)
      │
      ▼
[ profiles ] ──(user_id)──▶ [ accounts ] (Tenants / Workspaces)
                                │
   ┌────────────────────────────┼────────────────────────────┐
   ▼                            ▼                            ▼
[ contacts ]             [ conversations ]             [ whatsapp_config ]
(account_id)              (account_id)                 (account_id, UNIQUE)
```

### Key Multi-Tenancy Invariants:
1. **Account Foundation (`017_account_sharing.sql`):**
   * Every user signup automatically provisions an entry in `accounts` and designates the creator as `owner`.
   * Every operational table (`contacts`, `conversations`, `messages`, `pipelines`, `broadcasts`, `automations`, `flows`, `whatsapp_config`) enforces `account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE`.
2. **Row-Level Security (RLS):**
   * PostgreSQL RLS is enabled on 100% of public tables.
   * Access is gated via `is_account_member(account_id, min_role)` function, ensuring Tenant A can never query or mutate Tenant B's data.
3. **Role-Based Access Control (RBAC):**
   * `owner`: Full control, billing, ownership transfer, member management.
   * `admin`: Manage WhatsApp configuration, templates, team members, integrations.
   * `agent`: Read/write contacts, manage conversations, execute manual broadcast sends.
   * `viewer`: Read-only access to CRM data.

---

## 3. WhatsApp Cloud API & The Multi-Tenant Webhook Lifecycle

### How Multi-Tenant Routing Works:
```
Customer WhatsApp Message
          │
          ▼
Meta WhatsApp Cloud API (POST)
          │  (signed with x-hub-signature-256)
          ▼
/api/whatsapp/webhook (Your Next.js Route)
          │
          ├─ 1. Verify HMAC-SHA256 signature against META_APP_SECRET
          │     (Rejects 401 if invalid or missing)
          │
          ├─ 2. Inspect value.metadata.phone_number_id
          │
          ├─ 3. Query: SELECT * FROM whatsapp_config WHERE phone_number_id = ?
          │     (Resolves the exact tenant account_id)
          │
          ├─ 4. Upsert Contact & Conversation scoped to resolved account_id
          │
          ├─ 5. Insert Message row
          │
          ├─ 6. Mirror inbound media to Supabase chat-media bucket
          │
          └─ 7. Trigger Automations & AI Auto-reply
```

### Commercial SaaS: Tech Provider vs BYO-App
* **The SaaS Standard (Embedded Signup):** The platform operator (SydxAI) maintains **one Meta Developer App** with **one `META_APP_SECRET`**. Tenants onboard by clicking "Login with Facebook" (Embedded Signup) to connect their WhatsApp numbers to the platform app. All incoming webhooks arrive signed with the platform's secret and are routed by `phone_number_id`.
* **Bring-Your-Own-App (BYO):** If individual tenants configure their own Meta Developer Apps, their webhook URL must include their tenant identifier (e.g. `/api/whatsapp/webhook/[accountId]`) to verify against their respective secret stored encrypted in `whatsapp_config`.

---

## 4. 100% Serverless Deployment Guide (Vercel + Supabase)

### Why Serverless?
* **Zero Server Ops:** No SSH, Docker, Nginx, or Certbot maintenance.
* **Cost Efficiency:** $0/mo on Vercel Hobby + Supabase Free.
* **Auto-Scaling:** Automatically handles traffic spikes without memory crashes.

### Step 1: Push Code to GitHub
Ensure your repository is pushed to your GitHub account (`https://github.com/sai8341/wacrm`).

### Step 2: Configure Vercel Project
1. Log into [Vercel Dashboard](https://vercel.com) and click **Add New... ➔ Project**.
2. Select your repository `sai8341/wacrm` and click **Import**.
3. In **Environment Variables**, supply:

| Variable | Description | Value |
| :--- | :--- | :--- |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase Project URL | `https://<your-project-ref>.supabase.co` |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Public Supabase Anon Key | `eyJhbGciOi...` |
| `SUPABASE_SERVICE_ROLE_KEY` | Server-side master key | `eyJhbGciOi...` |
| `ENCRYPTION_KEY` | AES-256 token encryption (64 hex) | `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` |
| `META_APP_SECRET` | Meta App Secret from Meta Developer Portal | `<your-32-char-app-secret>` |
| `NEXT_PUBLIC_SITE_URL` | Production URL | `https://crm.sydxai.com` |
| `AUTOMATION_CRON_SECRET` | Secret protecting the automation cron | (Generate random hex string) |

4. Click **Deploy**.

### Step 3: Domain & Webhook Setup
1. In Vercel Project Settings ➔ **Domains**, add `crm.sydxai.com`.
2. Update DNS CNAME or A records as indicated by Vercel.
3. In [Meta Developer Portal](https://developers.facebook.com/apps):
   * Go to **WhatsApp ➔ Configuration ➔ Webhook**.
   * Callback URL: `https://crm.sydxai.com/api/whatsapp/webhook`
   * Verify Token: (The token saved in your CRM Settings ➔ WhatsApp)
   * Subscribe to: `messages`, `message_template_status_update`, `message_template_quality_update`, `message_template_components_update`.

### Step 4: Automations Cron Scheduling (Serverless Best Practice)
The CRM contains scheduled automations and "Wait" steps that park executions in `automation_pending_executions`.
To drain these on a serverless architecture without paid Vercel Pro crons:
1. Register on a free monitoring service like **[Cron-job.org](https://cron-job.org)** or **Better Stack**.
2. Create a 1-minute recurring HTTP job:
   * **URL:** `https://crm.sydxai.com/api/automations/cron`
   * **Method:** `GET`
   * **Header:** `x-cron-secret: <your_AUTOMATION_CRON_SECRET>`
   * **Frequency:** Every 1 minute (`* * * * *`)

---

## 5. Security & Maintenance Best Practices

1. **Keep Secrets Out of Client Code:** Never prefix service role keys, Meta secrets, or encryption keys with `NEXT_PUBLIC_`.
2. **Never Rotate `ENCRYPTION_KEY`:** Rotating `ENCRYPTION_KEY` orphans all previously encrypted WhatsApp access tokens in the database.
3. **Database Security:**
   * Periodically check Supabase Security Advisors (`supabase` MCP ➔ `get_advisors`).
   * Ensure `SECURITY DEFINER` functions in PostgreSQL always pin `SET search_path = ''`.
4. **Cloudflare Proxying:** For maximum security, proxy `crm.sydxai.com` through Cloudflare Free tier to benefit from unmetered Layer 3/4/7 DDoS protection and Bot Fight Mode.
