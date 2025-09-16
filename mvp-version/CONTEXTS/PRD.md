# GTM Dashboard — High-Level PRD (Production SaaS, simplified)

## 1) Summary

GTM Dashboard is a web app that brings GTM data (Google Analytics 4, Google Search Console, Google Ads, HubSpot, Instantly, Phantom Buster, etc.) into **BigQuery** for each customer (tenant). Users sign up and authenticate with **Supabase**, pay with **Stripe**, connect their tools, ask questions in plain English, and get charts + short insights. We keep the architecture **lean**: FastAPI based integration service on **Cloud Run** (with Microservices per tool), **Supabase Postgres** for app records, **BigQuery** for analytics tables and **materialized views**. No dbt, no Pub/Sub/Workflows/Scheduler/Jobs unless we hit a wall.

> Note: *Materialized views* are precomputed query results stored in BigQuery. They refresh automatically when referenced tables change, making repeated charts fast without extra infra.

---

## 2) Goals

* **Single source of truth**: All GTM data per tenant in BigQuery.
* **Conversational insights**: Ask questions → safe SQL → accurate answers.
* **Auto charts**: Generate chart specs for the frontend; allow pinning.
* **Sane MVP ops**: Minimal moving parts; defaults that work “out of the box”.

---

## 3) In Scope

* **Auth & User Store:** Supabase Auth (Google OAuth + email/password).
* **Billing:** Stripe Checkout + Billing Portal; one paid plan per tenant.
* **Tenant Isolation:** One BigQuery dataset per tenant (`tenant_id`); permissions scoped.
* **Integrations (no Airbyte):**
  * Direct **simple backend ingestion** for each tool.
  * FastAPI based microservices per tool.
  * Flow: User connects tool (supplies credentials) → we call the tool's API with the credentials → we transform raw data and we write to BigQuery.
* **Analytics Engine:** Google **ADK** agent service on Agent Engine (schema read, SQL generation, DRY-RUN/cost cap, execution).
* **Visualization:** Frontend (Next.js + Recharts) renders a compact **ChartSpec JSON**; pin/unpin charts stored in Supabase.
* **BigQuery Materialized Views:** Set up for pinned or frequently used queries.
* **Manual refresh:** A button in UI triggers a one-shot integration fetch (no internal scheduler).

---

## 4) Out of Scope (now)

* dbt; Pub/Sub; Cloud Workflows; Cloud Scheduler; Cloud Run Jobs (unless a blocker arises).
* Instantly integration.
* Multi-user roles per tenant.
* BI Engine (we’ll skip it to reduce setup complexity).

---

## 5) Architecture (very high level)

* **Frontend (Next.js + Recharts)**: Supabase client for auth; calls our API with a tenant-scoped JWT; renders charts from ChartSpec.
* **Tool Integration API (Cloud Run, FastAPI)**:

  * **Gateway** + modules for **Integrations**, **Analytics/Agent**, **Charts**, **Billing webhook**, **Tenant provisioning**.
  * Talks to **Supabase Postgres** (app records) and **BigQuery** (read/write).
* **Supabase Postgres (managed by Supabase)**: tenants, users, subscriptions, integrations, pins, saved queries.
* **BigQuery**: source tables per integration; **materialized views** for curated analytics; **CTE/SELECT-only** access for the agent.
* **Stripe**: hosted Checkout + Billing Portal; single webhook endpoint in API.

---

## 6) Data & Records

**BigQuery (per tenant dataset)**

* Landing tables: `ga4_*`, `gsc_*`, `google_ads_*`, `hubspot_contacts`, `hubspot_deals`, `hubspot_engagements` (names kept simple).
* Read layer: **SQL views** and **materialized views** for common queries:

  * `mv_daily_sessions_by_channel`
  * `mv_ads_spend_by_campaign`
  * `mv_search_queries_by_page`
  * `mv_funnel_by_channel`
* The agent **only queries allow-listed views/MVs** (not raw tables).

**Supabase Postgres (app records; no Cloud SQL)**

* `tenants` (id, name/slug, bq\_dataset\_ref, stripe\_customer\_id, stripe\_subscription\_id, created\_at)
* `users` (Supabase-managed) + `tenant_memberships`
* `integrations` (source, status, config json, last\_sync\_at, last\_error)
* `pins` (chart\_spec json, mv\_or\_sql\_ref, created\_at, tenant\_id)
* `saved_queries` (optional; sql text, label, created\_at)

---

## 7) APIs Needed

**Belongs to the API Microservices on Cloud Run**

1. **Auth & Tenant**

   * Verify Supabase JWT; resolve current tenant.
   * Provision tenant: create a dedicated BigQuery dataset for the tenant with appropriate permissions, store record in Supabase.

2. **Billing**

   * Stripe webhook: on successful checkout → mark subscription active for tenant; on cancel → mark inactive.

3. **Integrations**

   * Connect HubSpot: store OAuth/token (in Supabase **encrypted**) and save config.
   * Connect Instantly: store API key (in Supabase **encrypted**) and save config.
   * Connect Phantom Buster: store API key (in Supabase **encrypted**) and save config.
   * Manual refresh (per source): run a **one-shot** fetch.
   * Status: return basic health (connected?, last\_sync\_at, last\_error).

4. **Analytics / Agent**

   * Accept a user question; perform schema introspection on **allow-listed** views; generate **SELECT-only** SQL; **DRY-RUN**; execute; if large, write to a **temp table or materialized view**; return **ChartSpec JSON** + small sample.
   * ADK based agent service on Agent Engine.
   * Optionally return a link/reference to the MV for dashboard use.

5. **Charts & Dashboard**

   * Save/Delete pins (stores ChartSpec and MV/SQL reference).
   * List pins for dashboard.

---

## 8) User Flow

1. **Sign up** with Supabase (Google or email/password).
2. **Pay** via Stripe Checkout; upon webhook confirmation the tenant is marked active.
3. **Provision** automatically creates the tenant’s BigQuery dataset.
4. **Connect sources**

   * Instantly/Phantom Buster: integration setup on the frontend; data starts to land in BigQuery (no in-house scheduler).
   * HubSpot: click “Connect” → authorize → we store token; **manual “Sync Now”** pulls core objects (contacts, deals, engagements) into BigQuery.
5. **Ask a question** in chat → agent plans safe SQL → returns chart + insight.
6. **Pin** useful charts → they appear on the dashboard.
7. **Manual refresh** when needed.

---

## 9) Constraints, Performance & Security (MVP-friendly)

* **Performance**

  * BigQuery handles large scans efficiently by default; we add **DRY-RUN** to avoid surprises.
  * Materialized views speed up repeated dashboard queries **without** extra systems.
  * Recharts renders only **aggregated/bucketed** series; we never push huge row sets to the browser.

* **Scalability**

  * Cloud Run autoscales the API Microservices by default; no tuning needed for early usage.
  * BigQuery scales transparently; each tenant has its own dataset to avoid cross-tenant contention.
  * Provider-managed transfers remove our need to run schedulers.

* **Security**

  * Supabase Auth issues JWT; we read tenant context on each request.
  * BigQuery dataset-per-tenant with service account access scoped per dataset.
  * Agent runs **SELECT-only** against allow-listed views; no write/DDL/DML permissions.
  * Secrets/tokens stored encrypted in Supabase (keep scope minimal).
  * HTTPS by default on Cloud Run and Stripe.

> Why achievable “out of the box”?
>
> * Cloud Run (managed), BigQuery (serverless), Supabase (managed Postgres + Auth), Stripe (hosted) all provide production defaults—no servers, no queues, no schedulers for us to manage. This minimizes time spent on infra and lets us focus on integrations and the chat experience.

---

## 10) Acceptance Criteria

* New user can sign up (Supabase), pay (Stripe), and gets a BigQuery dataset created automatically.
* Tools connected and visible in the Integrations screen with **last sync** info.
* Tools “Connect” works; **Manual Sync** ingests data into BigQuery.
* Chat question “Top traffic sources last 30 days?” returns an accurate chart + short insight.
* User can **pin** a chart; dashboard lists pins; pin references an MV/SQL.
* Agent logs show **DRY-RUN** before execution; queries are **SELECT-only**; agent cannot access raw tables outside the allow-list.
* No cross-tenant data access is possible (spot-check with two test tenants).

---

## 11) Open Assumptions / Notes

* HubSpot rate limits are handled by simple backoff inside the single API call for “Manual Sync”. If this becomes slow for many tenants, we may add **Cloud Run Jobs** later (only if necessary).
* **BI Engine** is **not used** now to keep setup simple; materialized views give us enough speed for repeated charts.
* If HubSpot’s native BQ integration becomes essential and *requires* GCS as a staging area, we will add a minimal bucket then; for now we **avoid GCS** by using our own HubSpot pull.

---

## 12) Team Ownership (minimal handoffs)

* **Frontend:** Next.js app, Chat, Dashboard, Integrations UI, Recharts.
* **API/Backend:** Cloud Run services (Gateway + Integrations + Agent + Charts + Billing); BigQuery DDL (tables, views, materialized views); HubSpot pull logic.
* **Data Admin:** Create/maintain allow-listed views & materialized views per tenant (SQL files in repo).
* **Ops (shared):** Supabase project, Stripe product/price, Cloud Run services, BigQuery IAM templates.

---
