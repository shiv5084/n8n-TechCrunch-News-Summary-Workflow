# Render Free Tier Deployment Plan: n8n News Summary Workflow

This document provides a comprehensive, step-by-step guide to self-host and deploy **n8n** on the **Render Free Tier** and run the `TechCrunch_summary_workflow.json` workflow.

---

## ⚡ Executive Summary of Render Free Tier Limitations

While Render's Free Tier is excellent for hosting applications, it introduces two critical limitations that we must bypass for n8n to run reliably:

1. **Non-Persistent Disk**: Free web services do not support persistent disks. Any local SQLite database (default for n8n) will be wiped every time the container restarts (at least once a day, or on code changes).
   * **Solution**: Connect n8n to a free external cloud PostgreSQL database (e.g., Neon or Supabase).
2. **Inactivity Sleep (Spin-down)**: Render free services automatically spin down (go to sleep) after 15 minutes of inactivity. When asleep, **n8n's internal Schedule Triggers will NOT fire**.
   * **Solution A (Recommended)**: Set up an external cron service (like [cron-job.org](https://cron-job.org)) to ping n8n's health endpoint every 10–14 minutes to keep it awake.
   * **Solution B**: Replace n8n's internal `Schedule Trigger` with a `Webhook` node and trigger it externally using an external cron service.

---

## 🛠️ Step 1: Prepare Your Git Repository

To deploy on Render, you need a private GitHub (or GitLab) repository containing a `Dockerfile` that tells Render to run n8n.

1. **Create a new private repository** on GitHub (e.g., `n8n-render-deployment`).
2. **Add a `Dockerfile`** to the root of the repository with the following content:
   ```dockerfile
   FROM n8nio/n8n:latest
   ```
3. **Add a `.gitignore`** to ensure you don't commit credentials or local logs:
   ```text
   node_modules/
   .env
   .n8n/
   ```
4. Commit and push your code to your repository.

---

## 🗄️ Step 2: Set Up a Free PostgreSQL Database

Since Render's free database expires after 90 days, we recommend using a permanent free-tier PostgreSQL provider like **Neon.tech** or **Supabase**.

### Option A: Neon.tech (Recommended & Easiest)
1. Sign up for a free account at [Neon.tech](https://neon.tech).
2. Create a new project and database.
3. Copy the connection parameters from the **Connection String** shown in the dashboard.
   * **Note on Pooled vs. Direct connection**: By default, Neon shows a pooled connection string with a `-pooler` suffix in the host (e.g., `ep-sparkling-paper-a7eatsig-pooler.region.neon.tech`). Since n8n is a long-running app that maintains persistent database connections, you **should use the Direct connection** instead to prevent connection issues. 
   * **To use a Direct connection**: In your host parameter, simply remove the `-pooler` suffix (e.g., use `ep-sparkling-paper-a7eatsig.region.neon.tech` instead of `ep-sparkling-paper-a7eatsig-pooler.region.neon.tech`). You can also toggle off "Pooled connection" in the Neon dashboard to see the direct credentials.

### Option B: Supabase
1. Sign up at [Supabase](https://supabase.com).
2. Create a new project and wait for the database to provision.
3. Go to **Project Settings** -> **Database** and copy the **URI Connection String** under "Connection strings" (remember to replace `[YOUR-PASSWORD]` with your actual database password).

---

## 🚀 Step 3: Deploy n8n on Render

1. Log in to [Render Dashboard](https://dashboard.render.com).
2. Click **New +** and select **Web Service**.
3. Connect your GitHub repository containing the `Dockerfile`.
4. Configure the Web Service settings:
   * **Name**: `n8n-news-digest` (or any custom name)
   * **Region**: Choose the region closest to you
   * **Branch**: `main`
   * **Runtime**: `Docker`
   * **Instance Type**: `Free`
   * **Health Check Path**: **Leave this completely blank** (Do NOT set it to `/healthz` or `/`)
     > [!IMPORTANT]
     > Neon PostgreSQL databases scale to zero when idle and take a few seconds to wake up. If you set a Health Check Path in Render, Render will hit n8n, receive a `503 Database is not ready!` response, and immediately terminate the container (`SIGTERM`) before n8n has time to boot up. Leaving the health check path blank causes Render to use a simple TCP port check, giving n8n plenty of time to connect in the background.
5. Click **Advanced** to add the following **Environment Variables**:

| Variable Name | Value / Description | Importance |
| :--- | :--- | :--- |
| `PORT` | `10000` | Render expects the app to listen on this port |
| `N8N_PORT` | `10000` | Configures n8n to listen on the correct port |
| `N8N_ENCRYPTION_KEY` | *Your generated 32-character secure random string* | **CRITICAL**: Keeps credential decryption working across restarts |
| `DB_TYPE` | `postgresdb` | Tells n8n to use PostgreSQL instead of SQLite |
| `DB_POSTGRESDB_HOST` | *Your Neon/Supabase host (e.g., ep-sparkling-paper-a7eatsig.ap-southeast-2.aws.neon.tech (no -pooler).)* | Host of your external persistent database |
| `DB_POSTGRESDB_PORT` | `5432` | Port of your external database |
| `DB_POSTGRESDB_DATABASE` | *Your database name (e.g., neondb)* | Name of your database |
| `DB_POSTGRESDB_USER` | *Your database user* | Username of your database |
| `DB_POSTGRESDB_PASSWORD` | *Your database password* | Password of your database |
| `DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED` | `false` | Prevents connection issues with SSL-enabled databases like Neon/Supabase |
| `WEBHOOK_URL` | `https://your-service-name.onrender.com/` | **CRITICAL**: Set to your Render app URL for webhooks and OAuth callbacks |
| `N8N_SECURE_COOKIE` | `false` | Resolves cookie/session issues behind Render's load balancer |
| `NODE_OPTIONS` | `--max-old-space-size=360` | **MEMORY OPTIMIZATION**: Prevents Out-Of-Memory (OOM) crashes by forcing Node.js garbage collection to run aggressively before reaching the 512MB limit |
| `EXECUTIONS_DATA_PRUNE` | `true` | **MEMORY OPTIMIZATION**: Automatically deletes old execution data to keep database and memory usage low |
| `EXECUTIONS_DATA_MAX_AGE` | `24` | **MEMORY OPTIMIZATION**: Keeps only the last 24 hours of execution history |
| `DB_POSTGRESDB_CONNECTION_TIMEOUT` | `30000` | **NEON TIMEOUT OPTIMIZATION**: Gives Neon database enough time (30 seconds) to wake up from its sleep/cold start state on first boot |
| `DB_PING_TIMEOUT_MS` | `20000` | **NEON TIMEOUT OPTIMIZATION**: Increases connection check timeouts to prevent premature n8n container crashes during database startup |

6. Click **Deploy Web Service**. Render will build the Docker container and start n8n.

---

## ⏰ Step 4: Keep n8n Awake (Bypassing Inactivity Sleep)

To ensure your scheduled newsletter runs reliably every day, you must prevent n8n from sleeping.

1. Go to [cron-job.org](https://cron-job.org) and create a free account.
2. Click **Create Cronjob**.
3. Configure the settings:
   * **Title**: `Keep n8n awake`
   * **Address**: `https://your-service-name.onrender.com/healthz` (replacing with your Render Web Service URL)
   * **Schedule**: `Every 12 minutes` (Render sleeps after 15 minutes, so 12 minutes guarantees it stays active)
4. Click **Create**. This cronjob will constantly ping n8n's health API, ensuring it stays online 24/7.

> [!NOTE]
> The workflow now includes a **Webhook** node by default!
> 
> * **Public Webhook URL**: `https://your-service-name.onrender.com/webhook/trigger-news-digest`
> * **How it behaves**: Anyone opening this URL in their browser will instantly trigger your workflow on Render (even if the container was sleeping, Render will boot it up to handle the request).
> * **Response**: When clicked, visitors will see: *"Workflow Triggered Successfully! The TechCrunch AI Digest newsletter is being compiled and sent."*
> * **No Sleep Requirement**: If you share this webhook, you do **not** need to keep your n8n container awake 24/7. It can sleep, and it will only wake up when someone clicks the link to trigger the workflow.

---

## 📥 Step 5: Import and Configure the Workflow

Once n8n is deployed and accessible via your Render URL:

1. Open your web browser and navigate to `https://your-service-name.onrender.com`.
2. Set up your admin owner account (email and password).
3. In the left-hand sidebar, click **Workflows** -> **Add Workflow** (or click the top-right menu icon and select **Import from File**).
4. Select [TechCrunch_summary_workflow.json](file:///d:/N8N%20+%20Cursor%20workflow/n8n%20news%20summary%20workflow/TechCrunch_summary_workflow.json) from this project folder.
5. Set up credentials:
   * **Google Gemini Chat Model / OpenAI Chat Model**: Click on the respective node and configure your API key credential.
   * **Gmail (OAuth2)**:
     * **Do not upload the `client_secret_*.json` file**: n8n does not read credential files from the file system on Render; they are entered in the UI and stored encrypted in your database (Neon).
     * **Update Redirect URIs**: Go to the Google Cloud Console -> APIs & Services -> Credentials. Edit your OAuth 2.0 Client ID and add the following URL under **Authorized redirect URIs**:
       `https://your-service-name.onrender.com/rest/oauth2-credential/callback`
       *(Make sure to replace `your-service-name` with your actual Render service subdomain).*
     * **Link Account**: Open the Gmail node in the n8n UI, select **Create New Credential**, copy/paste the **Client ID** and **Client Secret** from your local `client_secret_*.json` file, and click **Sign in with Google** to complete the authentication consent flow.
6. Toggle the workflow status in the top right from **Inactive** to **Active** to start automatic executions!
