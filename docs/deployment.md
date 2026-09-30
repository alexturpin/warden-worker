# Deployment

## This repository's production deployment

Production is `https://vault.alexturpin.com`, deployed as `warden-worker` in
Cloudflare account `d85a0daa3e2452f1bcc7ba56d59ceb5e`.
`wrangler.toml` contains the Custom Domain, D1 database `warden-vault`, and the
attachment KV namespace ID. These IDs are configuration, not credentials.
Wrangler manages the Custom Domain's DNS record and certificate during deployment.

Install the pinned local CLI with `npm ci`. Local commands use the `personal`
OAuth profile explicitly:

```bash
npm run deploy
npm run db:migrate
npm run dev
npm exec -- wrangler secret list --profile personal
```

To verify the account, activate the directory's profile first (`whoami` does not
accept `--profile`):

```bash
npm exec -- wrangler auth activate personal
npm exec -- wrangler whoami
```

If OAuth expires, run `npm exec -- wrangler auth create personal`, then `whoami`
again. Activation selects an existing profile; it does not reauthenticate it.

The production database is initialized from `sql/schema.sql`; migrations included
in that snapshot are recorded in `d1_migrations`. Subsequent deployments apply new
migrations before publishing the Worker. Do not apply the snapshot to an existing
vault or rerun the initial migration-marking step against an existing database.

### Automatic deployments

The `Deploy production` GitHub Actions workflow deploys pushes to `main` and can
also be run manually. Deployments run serially. It downloads the pinned Web Vault,
builds Rust/WASM, applies D1 migrations, seeds equivalent domains, publishes the
Worker, and checks the website, `/api/alive`, and `/api/config`.

One GitHub environment secret is required: **`CLOUDFLARE_API_TOKEN`**, in the
`production` environment. The deployment job explicitly selects that environment.
Create the token at
[Cloudflare API Tokens](https://dash.cloudflare.com/profile/api-tokens), scoped to
the account above and the `alexturpin.com` zone, with:

- Account → Workers Scripts → Edit (deployment)
- Account → D1 → Edit (migrations and domain seeding)
- Account → Account Settings → Read (account discovery)
- Zone → Zone → Read (zone discovery)
- Zone → Workers Routes → Edit (Custom Domain management)

Add it in [the repository's production environment](https://github.com/alexturpin/warden-worker/settings/environments).
Do not copy the local OAuth token into CI. CI uses the API token directly and does
not pass `--profile personal`: that profile exists only on your machine.

KV and D1 are accessed through bindings at runtime. The pre-created KV namespace
ID means production CI does not create namespaces or read/write their contents,
so it does not require KV Edit. Account/database IDs are already committed in
configuration; production deployment does not require ID secrets.

After the configuration changes reach `main`, enable Actions for this fork if
GitHub displays the fork-workflow prompt, then run `Deploy production` or push a
commit to `main`. Do not also connect Workers Builds: that would deploy twice.

JWT signing secrets are stored on the Worker and preserved across deployments;
they are not GitHub secrets. Registration starts closed: `ALLOWED_EMAILS` is set
to `registration-disabled` (which matches no valid email), and the Web Vault hides
registration by default. To open registration for chosen addresses:

```bash
# Enter exact emails separated by commas, or an email glob such as *@alexturpin.com.
npm exec -- wrangler secret put ALLOWED_EMAILS --profile personal
# Enter false to display registration in the Web Vault.
npm exec -- wrangler secret put DISABLE_USER_REGISTRATION --profile personal
```

Changing `ALLOWED_EMAILS` affects future registrations; existing accounts remain
usable. JWT keys should stay stable across deployments to preserve sessions.
Attachments use KV (maximum 25 MB per file). Mobile push is not configured.

The optional backup workflow is separate: it needs a configured S3 or WebDAV
backend and `CLOUDFLARE_ACCOUNT_ID` / `D1_DATABASE_ID` repository secrets. D1's
built-in Time Travel is available, but offsite backups have not been configured.

The sections below document upstream alternatives. Their placeholder IDs and
optional development environment are not used by this production deployment.


This page covers the available deployment paths. Pick the one that fits your workflow and infrastructure.

- **CLI** — build and `wrangler deploy` from your machine.
- **GitHub Actions** — build and deploy from CI on every push; the same Cloudflare API token also drives the daily D1 backup workflow.
- **Cloudflare Workers Builds** — connect the repo in the Cloudflare dashboard so `git push` builds and deploys on Cloudflare.

## CLI Deployment

1. **Clone the repository:**

   ```bash
   git clone https://github.com/your-username/warden-worker.git
   cd warden-worker
   ```

2. **Create a D1 Database:**

   ```bash
   wrangler d1 create warden-db
   ```

3. **(Optional) Enable R2 Bucket for Attachments:**

   Warden uses KV for attachments storage by default. If you want to use R2 as storage backend:

   ```bash
   # Create the production bucket
   wrangler r2 bucket create warden-attachments
   ```

   Then enable the R2 binding in `wrangler.toml` by uncommenting the R2 bucket configuration sections.

   **Note:** Attachments are optional. If you remove both KV and R2 bindings, attachment functionality will be disabled but all other features will work normally.

4. **Configure your Database ID:**

   Set `database_id` directly in the production `[[d1_databases]]` section of
   `wrangler.toml` to the ID printed by `wrangler d1 create`. Database IDs are not
   secrets. Wrangler does not interpolate `${D1_DATABASE_ID}` in TOML; setting a
   shell variable or `.env` alone does not substitute a placeholder.

5. **Download the frontend (Web Vault):**

   ```bash
   # Default pinned version (override by exporting BW_WEB_VERSION)
   BW_WEB_VERSION="${BW_WEB_VERSION:-v2026.6.4}"
   if [ "${BW_WEB_VERSION}" = "latest" ]; then
     BW_WEB_VERSION="$(curl -s https://api.github.com/repos/dani-garcia/bw_web_builds/releases/latest | jq -r .tag_name)"
   fi

   # Download and extract
   wget "https://github.com/dani-garcia/bw_web_builds/releases/download/${BW_WEB_VERSION}/bw_web_${BW_WEB_VERSION}.tar.gz"
   tar -xzf "bw_web_${BW_WEB_VERSION}.tar.gz" -C public/
   rm "bw_web_${BW_WEB_VERSION}.tar.gz"

   # Remove large source maps to satisfy Cloudflare static asset per-file limits
   find public/web-vault -type f -name '*.map' -delete
   ```

   **Optional:** Apply lightweight UI overrides to generate `public/web-vault/css/vaultwarden.css`:

   ```bash
   mkdir -p public/web-vault/css/ && cp public/css/vaultwarden.css public/web-vault/css/
   ```

6. **Set up database and deploy the worker:**

   ```bash
   # Only run once before first deployment
   wrangler d1 execute vault1 --file sql/schema.sql --remote
   # For migrations
   wrangler d1 migrations apply vault1 --remote

   # (Optional) Seed global equivalent domains into D1
   # This downloads Vaultwarden's global_domains.json by default.
   bash scripts/seed-global-domains.sh --db vault1 --remote
   
   wrangler deploy
   ```

   This will deploy the worker and set up the necessary database tables.

7. **Set environment variables** as `Secret`

- `ALLOWED_EMAILS` your-email@example.com (supports glob patterns like `*@example.com`)
- `JWT_SECRET` a long random string
- `JWT_REFRESH_SECRET` a long random string

   **Optional mobile push relay settings:**  
     `PUSH_ENABLED=true`, `PUSH_RELAY_URI`, `PUSH_IDENTITY_URI` as text variables;  
     `PUSH_INSTALLATION_ID`, `PUSH_INSTALLATION_KEY` as secret variables.  
     See [Mobile Push Notifications](../README.md#mobile-push-notifications-optional) for more details.

8. **Configure your Bitwarden client:**

   In your Bitwarden client, go to the self-hosted login screen and enter the URL of your deployed worker.

   By default, the `*.workers.dev` domain is disabled, since it may throw 1101 error. It's highly recommended to use a custom domain instead; see [Configure Custom Domain](../README.md#configure-custom-domain-optional) for more details.

## CI/CD Deployment with GitHub Actions

This project includes GitHub Actions workflows for automated deployment. **This is the recommended default approach** for production: every push to `main` builds and deploys, and the same `CLOUDFLARE_API_TOKEN` secret also drives the daily [`Backup D1 Database`](/.github/workflows/backup-d1.yaml) workflow  — so a single token covers both deploy and backups.

### Required Secrets

For production deployment, add `CLOUDFLARE_API_TOKEN` to the `production` environment
(`Settings > Environments > production`). Optional backup/dev workflows use
repository secrets (`Settings > Secrets and variables > Actions`):

| Secret | Required | Description |
|--------|----------|-------------|
| `CLOUDFLARE_API_TOKEN` | yes | Your Cloudflare API token |
| `CLOUDFLARE_ACCOUNT_ID` | backups only | Account ID; production deploy reads the committed config |
| `D1_DATABASE_ID` | backups only | Production database ID; production deploy reads the committed config |
| `D1_DATABASE_ID_DEV` | no | Dev D1 database ID (required only if you use the `Deploy Dev` workflow on the `dev` branch) |

#### How to Get Your Cloudflare Account ID

1. Log in to the [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Select your account
3. Your Account ID is displayed in the right sidebar of the Overview page, or in the URL: `https://dash.cloudflare.com/<account-id>`

#### How to Get Your Cloudflare API Token

The `CLOUDFLARE_API_TOKEN` requires the following permissions:
- **Edit Cloudflare Workers**: Required for deploying the Worker
- **Edit D1**: Required for database migrations and backups
- **Edit KV**: Only needed when CI creates namespaces or accesses KV contents; runtime attachment access uses the binding

1. Visit [https://dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Click **Create Token**
3. Use the **Edit Cloudflare Workers** template
4. Add **Account** → **D1** → **Edit** under `Permissions`
5. Select `Account Resources` and `Zone Resources`
6. Click **Continue to Summary** and then **Create Token**

### Optional Variables

#### Web Vault frontend version

You can pin/override the bundled Web Vault (bw_web_builds) version via GitHub Actions Variables:

| Variable | Applies to | Default | Example | Notes |
|----------|------------|---------|---------|-------|
| `BW_WEB_VERSION` | prod (`main`) | `v2026.6.4` | `v2026.6.4` | Set to `latest` to follow upstream latest release |
| `BW_WEB_VERSION_DEV` | dev (`dev`) | `v2026.6.4` | `v2026.6.4` | Set to `latest` to follow upstream latest release |

#### Global Equivalent Domains

Bitwarden clients use `globalEquivalentDomains` for URI matching across well-known domain groups.

To avoid bundling a large JSON file into the Worker, the dataset can be stored in D1 and seeded during deploy.

| Variable | Applies to | Default | Example | Notes |
|----------|------------|---------|---------|-------|
| `SEED_GLOBAL_DOMAINS` | prod + dev | `true` | `false` | Set to `false` to skip seeding (API returns empty list) |
| `GLOBAL_DOMAINS_URL` | prod | (empty) | raw GitHub URL | Optional: pin a specific Vaultwarden tag/commit for reproducible deploys |
| `GLOBAL_DOMAINS_URL_DEV` | dev | (empty) | raw GitHub URL | Same as prod, but for dev workflow |

If you skip seeding, `/api/settings/domains` and `/api/sync` will return `globalEquivalentDomains: []`.

### Usage

1. **Fork or clone the repository** to your GitHub account

2. **Configure the required secrets** in your repository settings

3. **(Optional) Enable R2 bucket for attachments:**

   Warden uses KV for attachments storage by default. If you want to use R2 as storage backend:

   1. **Create R2 buckets in Cloudflare Dashboard before running the action:**
      - Go to **Storage & databases** → **R2** → **Create bucket**
      - Create a production bucket (e.g., `warden-attachments`)

   2. **Add the production binding to `wrangler.toml`:**

      ```toml
      [[r2_buckets]]
      binding = "ATTACHMENTS_BUCKET"
      bucket_name = "warden-attachments"
      ```

   R2 takes precedence over KV when both are configured. The production workflow
   deploys the committed bindings; it does not append a binding from `R2_NAME`.

4. **Manually trigger the `Deploy production` Action** from the GitHub Actions tab in your repository

5. **Monitor the deployment** in the Actions tab of your repository

6. **Set environment variables** as `secret` in the Cloudflare dashboard (following the command line deployment steps):
   - `ALLOWED_EMAILS` your-email@example.com (supports glob patterns like `*@example.com`, comma separated)
   - `JWT_SECRET` a long random string
   - `JWT_REFRESH_SECRET` a long random string
   - Optional mobile push settings:
     `PUSH_ENABLED=true`, `PUSH_RELAY_URI`, `PUSH_IDENTITY_URI`, `PUSH_INSTALLATION_ID`, `PUSH_INSTALLATION_KEY`.  
     See [Mobile Push Notifications](../README.md#mobile-push-notifications-optional) for more details.


> [!IMPORTANT]
> The server can't work without these three environment variables. If you forget to set them, the server will crash.

If you want to show a 'Create account' button in frontend, you can add `DISABLE_USER_REGISTRATION` as `text` and set it to `false`. Check [Environment Variables](../README.md#environment-variables) for more details.

By default, the `*.workers.dev` domain is disabled, since it may throw 1101 error. It's highly recommended to use a custom domain instead; see [Configure Custom Domain](../README.md#configure-custom-domain-optional) for more details.

## Cloudflare Workers Builds (Dashboard) — Optional Alternative

[Workers Builds](https://developers.cloudflare.com/workers/ci-cd/builds/) is Cloudflare's native Git integration: connect the repository once in the dashboard and every push builds and deploys the Worker on Cloudflare, without a GitHub Actions deploy run.

> [!NOTE]
> This is an **optional alternative** to the default [GitHub Actions](#cicd-deployment-with-github-actions) flow, not a replacement. Two things to weigh before adopting it:
> - **Backups still need a GitHub token.** The daily `Backup D1 Database` workflow (`.github/workflows/backup-d1.yaml`) authenticates with the `CLOUDFLARE_API_TOKEN` GitHub secret, independent of how you deploy. Adopting Workers Builds does **not** remove that secret — you'd maintain *two* credentials: the Cloudflare build token (deploy) and the GitHub token (backups).
> - **Avoid double-deploys.** If you connect Workers Builds, disable the GitHub Actions `Deploy production` workflow (Actions tab → **Deploy production** → **Disable workflow**, or remove its `push:` trigger) so `main` is not deployed twice.
> - **Slow deployment speed.** The Cloudflare build environment has low RAM, CPU, and the deployment will cost you 7 minutes or so, using up your Worker build time.

Because this is a Rust→WASM Worker (the Workers Builds image does not ship Rust) and the frontend, database id, and migrations are all resolved at build time, the pipeline is encapsulated in two scripts:

- [`scripts/cf-build.sh`](../scripts/cf-build.sh) (**Build** phase) — installs the Rust toolchain, downloads the Web Vault frontend, substitutes the D1 database id, and compiles the Worker.
- [`scripts/cf-deploy.sh`](../scripts/cf-deploy.sh) (**Deploy** phase) — bootstraps/migrates D1, optionally seeds global domains, then runs `wrangler deploy`.

**No separate `CLOUDFLARE_API_TOKEN` is needed for the deploy itself.** Workers Builds auto-generates a *build token* and uses it as the credential for the deploy phase. Because the D1 migrations run in that same deploy phase (`cf-deploy.sh`), they reuse that one token — you just grant it D1 access once (step 2 below). (The separate daily backup workflow still relies on the `CLOUDFLARE_API_TOKEN` GitHub secret — see the note above.)

### Setup

1. In the Cloudflare dashboard, go to **Workers & Pages → `warden-worker` → Settings → Build**, then **Connect** your GitHub/GitLab repository (production branch: `main`). This auto-generates the build token.

2. **Grant the build token D1 access** (one-time). The auto-generated token includes Workers Scripts / KV / R2 edit, but **not D1**, so migrations would fail with an authorization error. Go to [My Profile → API Tokens](https://dash.cloudflare.com/profile/api-tokens), edit the token named **"Workers Builds"**, and add a permission row: **Account → D1 → Edit**. (If you'd rather not run migrations in CI, skip this and set `SKIP_D1=1` in step 4 — see the note below.)

3. Set the build commands:

   | Field | Value |
   |-------|-------|
   | **Build command** | `bash scripts/cf-build.sh` |
   | **Deploy command** | `bash scripts/cf-deploy.sh` |

   > `cf-deploy.sh` prepends `$HOME/.cargo/bin` to `PATH` so that `wrangler deploy` can find `cargo`/`worker-build` (installed during the build phase) when it re-runs `wrangler.toml`'s `[build]` step.

4. Add **Build variables** (Settings → Build → *Variables and Secrets*). None are secrets — the deploy credential is the build token from step 2.

   | Name | Required | Description |
   |------|----------|-------------|
   | `D1_DATABASE_ID` | yes | Production D1 database id (substituted into `wrangler.toml`) |
   | `BW_WEB_VERSION` | no | `bw_web_builds` tag (default `v2026.6.4`); set `latest` to track upstream |
   | `WRANGLER_VERSION` | no | Pinned wrangler (default `4.82.1`) |
   | `WORKER_BUILD_VERSION` | no | Pinned worker-build (default `0.8.5`; match the `worker` dep in `Cargo.toml`) |
   | `R2_NAME` | no | R2 bucket name; enables the `ATTACHMENTS_BUCKET` binding |
   | `SEED_GLOBAL_DOMAINS` | no | `false` to skip seeding global equivalent domains |
   | `GLOBAL_DOMAINS_URL` | no | Pin a specific `global_domains.json` source |
   | `SKIP_D1` | no | `1` to skip all D1 bootstrap/migrate/seed steps (deploy still runs) |
   | `CLOUDFLARE_ACCOUNT_ID` | no | Set only if wrangler cannot infer the account during migrations |

5. Set the runtime **Worker** secrets (these are *not* build variables — set them under the Worker's **Settings → Variables and Secrets**, not the build config): `ALLOWED_EMAILS`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, plus any optional push settings. See step 7 of [CLI Deployment](#cli-deployment).

6. **Push to `main`.** Cloudflare runs `scripts/cf-build.sh` then `scripts/cf-deploy.sh`. Watch progress under the Worker's **Builds** tab.

> [!NOTE]
> The first build is slow (it compiles the Rust toolchain dependencies and `worker-build` from scratch). Subsequent builds reuse the build cache and are faster.

> [!NOTE]
> If you set `SKIP_D1=1` (or skip step 2), the Worker still builds and deploys, but D1 migrations are **not** applied automatically — apply them yourself when the schema changes (`npx wrangler d1 migrations apply vault1 --remote`), or run the GitHub Actions `Deploy production` workflow manually.

> [!IMPORTANT]
> The default `Deploy production` workflow deploys on every push to `main`. If you adopt Workers Builds, **disable that workflow** (repository **Actions** tab → select **Deploy production** → **Disable workflow**, or remove the `push:` trigger in `.github/workflows/push-cloudflare.yaml`) so `main` is not deployed twice. Leave the `Backup D1 Database` workflow enabled — it still runs on its own schedule.
