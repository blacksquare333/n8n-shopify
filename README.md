# Shopify AI Description Pipeline (n8n)

An automated data pipeline for e-commerce: on a daily schedule, it fetches product data from a Shopify store via Admin API, splits the JSON response into individual product items, generates SEO-friendly product descriptions with an LLM (DeepSeek), and writes the results back to Supabase.

Built with [n8n](https://n8n.io) (self-hosted via Docker).

## Workflow

```
Schedule Trigger → HTTP Request (Shopify) → Split Out → HTTP Request (DeepSeek) → Supabase (Update)
```

| Node | What it does |
|---|---|
| **Schedule Trigger** | Triggers the workflow automatically at 8 AM daily — no manual script execution needed |
| **HTTP Request (Shopify)** | Fetches product data from the target Shopify store via Admin API (`GET /admin/api/{version}/products.json`), authenticated with a Header Auth credential (`X-Shopify-Access-Token`) |
| **Split Out** | Splits the single JSON response into individual product items — the n8n equivalent of `for p in data['products']`. Item count = loop count for all downstream nodes |
| **HTTP Request (DeepSeek)** | Generates an SEO-friendly product description for each product via DeepSeek chat completions API (one API call per item) |
| **Supabase (Update)** | Writes title + AI description back to Supabase, matching rows on `shopify_id` |

## Key Design Notes

- **Idempotent runs**: uses Update (matched on `shopify_id`) instead of Create, so the workflow can run daily without violating unique constraints or producing duplicate rows. (The Supabase node in this n8n version lacks a native Upsert; for true upsert behavior, fall back to an HTTP Request node calling Supabase REST API with `Prefer: resolution=merge-duplicates`.)
- **Cost control**: `?limit=3` during development to cap LLM API calls while testing. Remove or raise for production.
- **Credentials**: all secrets (Shopify token, DeepSeek key, Supabase service key) are stored in n8n's credential store, not in the workflow JSON.

## Troubleshooting Notes (real issues hit during the build)
**1. Shopify 401 — legacy custom apps blocked by 2026 policy**
- Symptom: after fixing the store domain, requests still returned 401. As of January 1, 2026, merchants can no longer create new legacy custom apps; the new dev dashboard flow only issues Client ID/Secret (OAuth), not a `shpat_` Admin API token.
- Fix: on a development store, Settings → Apps still offers **"Allow legacy custom app development"**. After enabling it: create app → grant `read_products` scope → Install → copy the one-time `shpat_` token.

**2. Expression vs Fixed mode confusion**
- Symptom: switched modes by feel; got "failed to parse" errors, or expressions sent as literal text.
- Fix: the only rule that matters — **if the value contains `{{ }}` (dynamic data), use Expression; if it's a constant, use Fixed.** It has nothing to do with the field's purpose. The same node can mix both (e.g. Supabase Update: Condition operator = Fixed, Field Value with `{{ }}` = Expression).

## Quick Start

1. Import the workflow JSON into n8n
2. Set credentials (Shopify, DeepSeek, Supabase) in n8n's credential store
3. Adjust `?limit=` in the Shopify node for production vs. testing
4. Activate the workflow# n8n-shopify
