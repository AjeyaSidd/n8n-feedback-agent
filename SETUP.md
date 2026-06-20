# Setup Guide — n8n App Feedback Analyst Agent

This guide covers setting up your entire infrastructure to run the n8n App Feedback Analyst Agent. You will configure:
1. A **Supabase** instance (PostgreSQL database) for storing n8n data.
2. A **Render** Web Service running **n8n** in a Docker container.
3. A **Render** Web Service running the **Google Play Store Scraper API**.
4. Credentials and configuration inside **n8n**.

---

## 🛠️ Step 1: Create a PostgreSQL DB on Supabase (Free Tier)

n8n needs a database to persist users, workflows, execution logs, and credentials. Supabase provides a free PostgreSQL database.

1. Go to [Supabase](https://supabase.com) and sign up or sign in.
2. Click **New Project** and select your organization.
3. Configure the project:
   - **Name**: `n8n-db`
   - **Database Password**: *Generate a secure password and save it somewhere safe.*
   - **Region**: Choose a region close to your Render services (e.g., `us-east-1` or your preferred local region).
   - **Pricing**: Select the **Free** tier.
4. Click **Create new project** and wait for database provisioning to complete (takes 1-2 minutes).
5. Once ready, click on **Project Settings** (gear icon at the bottom left) -> **Database**.
6. Scroll down to **Connection Info** and copy the following parameters:
   - **Host** (e.g., `aws-0-us-east-1.pooler.supabase.com`)
   - **Database name** (usually `postgres`)
   - **Port** (`5432` or `6543` for connection pooling. Use `5432` for direct connection).
   - **Username** (usually `postgres`)

---

## 🚀 Step 2: Deploy n8n on Render (Docker Setup)

We will run the official n8n community edition inside a Docker container on Render.

1. Go to [Render](https://render.com) and log in.
2. Click **New +** at the top right and select **Web Service**.
3. Choose **Deploy an image from a registry**.
4. Enter the official n8n docker image path in the **Image URL** input:
   ```text
   docker.io/n8nio/n8n:latest
   ```
5. Click **Next** and configure the Web Service settings:
   - **Name**: `my-n8n-instance`
   - **Region**: Same region as your Supabase database.
   - **Runtime**: Docker (automatically selected based on the image URL)
   - **Instance Type**: Select **Free** (or Starter if you require persistent storage, though connecting to Supabase allows n8n to run statelessly on the Free tier!).
6. Expand the **Advanced** section and add the following **Environment Variables**:

| Environment Variable | Value | Description |
| :--- | :--- | :--- |
| `DB_TYPE` | `postgresdb` | Instructs n8n to connect to a PostgreSQL database. |
| `DB_POSTGRESDB_HOST` | *Your Supabase Host* | The host URL you copied from Supabase. |
| `DB_POSTGRESDB_PORT` | `5432` | The database connection port. |
| `DB_POSTGRESDB_DATABASE` | `postgres` | Default database name. |
| `DB_POSTGRESDB_USER` | `postgres` | Database admin user. |
| `DB_POSTGRESDB_PASSWORD` | *Your Supabase DB Password* | Password chosen during Supabase setup. |
| `N8N_ENCRYPTION_KEY` | *Generate a random long string* | Used to encrypt credentials inside your database. |
| `N8N_BASIC_AUTH_ACTIVE` | `true` | Enables basic user login authentication. |
| `N8N_BASIC_AUTH_USER` | `admin` | Your desired login username. |
| `N8N_BASIC_AUTH_PASSWORD`| *Set a strong login password* | Your desired login password. |
| `PORT` | `5678` | Tell Render to route external requests to port 5678. |

7. Scroll to the bottom and click **Create Web Service**. 
8. Once the build completes and says `User settings loaded from ...` in the logs, open the Render provided service URL (e.g., `https://my-n8n-instance.onrender.com`).
9. Log in using the `N8N_BASIC_AUTH_USER` and `N8N_BASIC_AUTH_PASSWORD` you configured.

---

## 🕵️ Step 3: Deploy the Play Store Scraper API on Render

Because n8n does not natively scrape Google Play Store reviews, we deploy a lightweight, open-source API wrapper that queries the Google Play Store on demand.

### Scraper Code Template

You will deploy a simple Node.js application to Render.
First, create a repository on GitHub (or use Render's quick deploy) containing the following two files:

#### `package.json`
```json
{
  "name": "google-play-scraper-api",
  "version": "1.0.0",
  "description": "Simple API wrapper for google-play-scraper",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "google-play-scraper": "^9.1.1"
  }
}
```

#### `server.js`
```javascript
const express = require('express');
const gplay = require('google-play-scraper');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/api/apps/:appId/reviews', async (req, res) => {
  const appId = req.params.appId;
  const num = parseInt(req.query.num) || 100;
  const lang = req.query.lang || 'en';
  const country = req.query.country || 'us';

  try {
    const results = await gplay.reviews({
      appId: appId,
      sort: gplay.sort.NEWEST,
      num: num,
      lang: lang,
      country: country
    });
    
    res.json({ results });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});
```

### Steps to Deploy on Render:
1. Create a private/public repository on GitHub with the files above.
2. On Render, click **New +** at the top right -> **Web Service**.
3. Link your GitHub repository.
4. Configure the Web Service:
   - **Name**: `google-play-api`
   - **Runtime**: `Node`
   - **Build Command**: `npm install`
   - **Start Command**: `npm start`
   - **Instance Type**: Select **Free**.
5. Click **Create Web Service** and wait for deployment.
6. Copy the deployed service URL (e.g., `https://google-play-api.onrender.com`). You will need this for the n8n `Config` node.

---

## 🔗 Step 4: Import and Configure the Workflow in n8n

Now that both Render services are online, let's configure the n8n workflow.

### 1. Import the JSON
1. Open your n8n dashboard.
2. Click **Workflows** on the left menu, then click **Add workflow** (or **New**).
3. Click the three dots (`...`) in the upper-right corner and select **Import from File**.
4. Select the [workflow.json](file:///c:/Users/Ajeya%20Siddhartha/Projects/n8n-feedback-agent/workflow.json) file from this repository.

### 2. Configure the `Config` Node
Double-click the first node in the workflow labeled **Config**. Update the following values:
- `reportEmail`: Input the email address you want to receive the weekly report (e.g., `yourname@gmail.com`).
- `scraperApiUrl`: Paste your Render Play Store Scraper URL (e.g., `https://google-play-api.onrender.com`) **without** a trailing slash.
- `apps`: Customize the list of apps you wish to monitor. For each app, specify:
  - `name`: Human-readable name.
  - `appStoreId`: App Store numeric ID from the app's App Store webpage URL (e.g. for `https://apps.apple.com/.../id1404871703`, ID is `1404871703`). Set to `null` if the app is Android-only.
  - `playStoreId`: Package name from the Google Play Store URL (e.g. `com.nextbillion.groww`). Set to `null` if the app is iOS-only.
  - `logoUrl`: An image link to the app's logo (typically from the Play Store icon) to display inside the email report.

---

## 🔑 Step 5: Configure Credentials in n8n

The workflow uses two external credentials which must be set up inside n8n:

### 1. Gemini Flash API Credentials
The n8n workflow talks directly to Google's Generative Language API.
1. Open the node named **Call LLM API** in n8n.
2. Click on the **Credential for HTTP Query Auth** dropdown.
3. Select **Create New Credential**.
4. Configure the Query Auth parameters:
   - **Name**: `key` (leave as default)
   - **Value**: *Your Gemini API Key* (generate one for free from Google AI Studio).
5. Save the credential.

### 2. Resend API Credentials
The workflow sends emails using the Resend service.
1. Sign up for a free account at [Resend](https://resend.com) and generate an API key.
2. Open the node named **Send report** in n8n.
3. Click on the **Credential for HTTP Custom Auth** dropdown.
4. Select **Create New Credential**.
5. Configure the Custom Auth parameters:
   - **Name**: `Authorization`
   - **Value**: `Bearer re_xxxxxxxxxxxxxxxxxxxx` (Your Resend API Key).
6. Save the credential.

---

## ⏰ Step 6: Testing & Production Scheduling

1. Click **Execute Workflow** at the bottom of the canvas to run a test.
2. Once successful, check the recipient inbox for the compiled **App Review Report** HTML email.
3. To automate this report, click the **Webhook** or **Schedule** node (or add a **Cron / Schedule Trigger** in place of the webhook/manual trigger) to set up a recurring schedule (e.g., every Monday at 9:00 AM).
4. Save and toggle the workflow **Active** (using the switch in the top right).
