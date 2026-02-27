# BotMilu- - Sync Architecture

This project implements a product synchronization system using **Supabase Native** technologies. It syncs products from the Aleph API and WooCommerce to a Supabase database.

## Architecture

1.  **Database**: PostgreSQL hosted on Supabase.
    - `products_data`: Stores merged product info (Price, Stock, Images).
    - `categories`: Stores hierarchical category structure (Rubro/Subrubro).
2.  **CMS (Content Management System)**: Payload CMS (Next.js App Router).
    - Connected natively to the same local Supabase Postgres database.
    - Reserved for specific custom collections and internal business management tooling.
3.  **Edge Function** (`sync-aleph`): A Deno/TypeScript function that performs the heavy lifting:
    - Fetches all products from Aleph.
    - Fetches real-time stock and WooCommerce images (concurrently).
    - Upserts data to the database.
    - Cleans up stale records (products no longer in the feed).
4.  **Scheduler**: `pg_cron` extension in the database triggers the Edge Function every hour via HTTP POST.

----

## Local Development

### Prerequisites

- [Supabase CLI](https://supabase.com/docs/guides/cli) installed.
- Docker (required for Supabase local dev).
- Git.

### 1. Setup Environment

Copy the example environment file and fill in your keys:

```bash
cp .env.example .env.local
```

_Note: `.env.local` is used for **local function testing**._

### 2. Start Local Supabase

This starts a complete Supabase stack locally (DB, Auth, Edge Runtime).

```bash
npx supabase start
```

### 3. Deploy Local Migrations

Create the tables and extensions in your local DB:

```bash
npx supabase migration up
```

### 4. Configure Self-Hosted Environment

For self-hosted instances (Docker/Coolify), you must set the `app.api_url` variable to your project's function URL. The default is `http://127.0.0.1:54321/functions/v1/sync-aleph`.

To change it:

```sql
ALTER DATABASE postgres SET app.api_url = 'http://your-production-url/functions/v1/sync-aleph';
```

### 5. Test the Edge Function Locally

You can invoke the function directly on your machine.

```bash
npx supabase functions serve --no-verify-jwt --env-file .env.local
```

```bash
curl -i --location --request POST 'http://127.0.0.1:54321/functions/v1/sync-aleph' \
  --header 'Content-Type: application/json'
```

### 6. Start Payload CMS Local Server

To interact with the Payload CMS admin panel connected to your local Supabase database:

```bash
cd payload-cms
npm run dev
```

Access the panel locally through `http://localhost:3000/admin`. The database connection is handled natively to your Supabase default Postgres port (`127.0.0.1:54322`).

---

## Deployment (Test & Production)

To deploy to a live Supabase project (Staging or Production), follow these steps:

### 1. Link Project

Link your local repo to the remote Supabase project:

```bash
npx supabase link --project-ref your-project-id
```

### 2. Set Production Secrets

The Edge Function uses Supabase Secrets (not .env files) in production.

```bash
npx supabase secrets set ALEPH_API_URL=http://aleph.dyndns.info/integracion/api
npx supabase secrets set ALEPH_API_KEY=...
npx supabase secrets set ALEPH_CLIENT_ID=...
npx supabase secrets set WC_API_URL=...
npx supabase secrets set WC_CONSUMER_KEY=...
npx supabase secrets set WC_CONSUMER_SECRET=...
```

### 3. Deploy Database Changes

Apply the table definitions and cron setup:

```bash
npx supabase migration up
```

### 4. Deploy Edge Function

Upload the `sync-aleph` function code:

```bash
npx supabase functions deploy sync-aleph
```

---

## Scripts and Testing

We have configured several npm scripts in `package.json` to make it easier to test the functions both locally in your Mac, and against your remote VPS (Production).

### 1. Sync All or Individual Entities

To test synchronization logic in your **Local Dev Environment**:

```bash
npm run sync:local:products
npm run sync:local:clients
npm run sync:local:comprobantes
```

To trigger synchronization directly into your **VPS (Production)**:

```bash
npm run sync:prod:products
npm run sync:prod:clients
npm run sync:prod:comprobantes
```

### 2. Test Client Purchases (`get-client-purchases`)

The `get-client-purchases` Edge Function fetches the complete purchase history for a specific client (e.g. `cliente_id=2519`), including all voucher items (`comprobantes_items`).

To test **locally**:

```bash
npm run test:purchases:local
```

To test against your **Production VPS**:

```bash
npm run test:purchases:prod
```

---

## Important Note on Edge Functions in Production (VPS)

> **⚠️ "Failed to retrieve edge functions"**

When using the self-hosted version of Supabase Studio (the panel in your VPS at `https://lmn.server.neuraz.io`), when you click on the "Edge Functions" tab, you will **always** see the warning:

`Failed to retrieve edge functions. Local functions can be found at supabase/functions folder.`

**This is normal and expected.** The Supabase Studio UI tries to connect to the commercial Supabase Cloud API to list functions. Since you are running a private Self-Hosted cloud, that API does not exist. However, the functions **are running properly in the background** via Docker (Deno Edge Runtime). You must use the `npm run sync:prod:...` commands or HTTP requests to interact with them as the visual explorer is not supported in self-hosted setups without cloud accounts.

---

## Configuration

### Sync Schedule

The sync schedule is managed dynamically via the `public.sync_config` table in the database. You do not need to edit migration files to change the schedule.

**To view current schedules:**

```sql
SELECT * FROM public.sync_config;
```

**To update a schedule (e.g., run clients sync every 6 hours):**

```sql
UPDATE public.sync_config
SET cron_expression = '0 */6 * * *'
WHERE collection = 'clients';
```

**To disable a sync:**

```sql
UPDATE public.sync_config
SET is_active = false
WHERE collection = 'products';
```

### Troubleshooting

- **Logs**: View function logs in the Supabase Dashboard > Edge Functions > sync-aleph > Logs.
- **Timeouts**: The function is designed with concurrency. If it times out, consider reducing batch size in the code.
