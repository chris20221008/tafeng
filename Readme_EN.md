# Tafeng

Tafeng is a WebSSH workspace prototype designed for Cloudflare Workers. It is built with React, Vite, and TypeScript. The project now uses a pure Worker architecture: static frontend assets, authentication, settings, connection management, command history, file APIs, and WebSocket terminal bridging are all handled by Cloudflare Worker.

Chinese documentation: [README.md](./README.md)

## Features

- WebSSH terminal workspace with a macOS Terminal-like interface.
- Save VPS connection profiles: IP/domain, port, username, password, or private key.
- Admin-password login on the home page.
- Optional two-factor authentication setting.
- Chinese and English UI.
- File listing, text config editor, upload and download API placeholders.
- Upload API designed for files up to 10 GB.
- Live CPU, memory, swap, disk usage, and process list panels.
- Global command history across all VPS connections, up to 100000 entries.
- Cloudflare Worker + KV + R2 deployment structure.
- Future real SSH/SFTP integration is expected to use Worker TCP Socket.

## Current Status

The project currently includes the frontend, Worker APIs, authentication, settings, connection management, command history, file APIs, and monitoring UI. Real SSH/SFTP protocol integration is isolated in [worker/sshBridge.ts](./worker/sshBridge.ts). The current implementation is a runnable development bridge.

The goal of Option A is to deploy only Cloudflare Worker, without a separate traditional backend. For real SSH support, implement the bridge in `worker/sshBridge.ts` using Cloudflare Workers' `cloudflare:sockets` TCP Socket API to connect to the VPS `22` port.

## Project Structure

```text
tafeng/
├── src/                 # React + Vite + TypeScript frontend
├── worker/              # Cloudflare Worker APIs and WebSocket service
├── shared/              # Shared frontend/Worker types
├── public/              # Static assets
├── dist/client/         # Frontend build output, not committed
├── wrangler.toml        # Cloudflare Worker config
├── package.json         # Frontend and Worker scripts
└── LICENSE              # MIT License
```

## Requirements

- Node.js 18 or newer. Node.js 20+ is recommended.
- npm 9 or newer.
- A Cloudflare account.
- Wrangler CLI. This project includes `wrangler` in devDependencies, so you can use `npm run worker:dev` and `npm run worker:deploy`.

## Deploy to Cloudflare Worker from the Web UI

This is the recommended deployment path for most users. Start by forking the repository on GitHub, then finish the deployment in the Cloudflare Dashboard. No local terminal commands are required for this path.

### 1. Fork the Project on GitHub

1. Open this project's GitHub repository page.
2. Click `Fork` in the top-right corner.
3. Choose your GitHub account or organization as the owner.
4. Keep the repository name as `tafeng`, or rename it if you prefer.
5. Click `Create fork`.

After that, you will have your own repository, for example:

```text
https://github.com/your-username/tafeng
```

All following changes should be made in your fork, not in the upstream repository.

### 2. Create KV Namespaces in Cloudflare

Tafeng uses KV to store sessions, settings, VPS connection profiles, and command history.

1. Open the [Cloudflare Dashboard](https://dash.cloudflare.com/).
2. Go to `Storage & Databases` in the sidebar.
3. Open `KV`.
4. Click `Create namespace`.
5. Create the production namespace:

```text
tafeng-kv
```

6. Create the preview namespace:

```text
tafeng-kv-preview
```

7. Open each namespace and copy its Namespace ID.

You need two values:

```text
Production KV ID
Preview KV ID
```

### 3. Create R2 Buckets in Cloudflare

Tafeng uses R2 for uploaded files and large-file staging.

1. In the Cloudflare Dashboard sidebar, open `R2 Object Storage`.
2. Click `Create bucket`.
3. Create the production bucket:

```text
tafeng-files
```

4. Create the preview bucket:

```text
tafeng-files-preview
```

The bucket names should match [wrangler.toml](./wrangler.toml). You may use different names, but then you must also update `wrangler.toml`.

### 4. Edit wrangler.toml on GitHub

Go back to your forked GitHub repository.

1. Open [wrangler.toml](./wrangler.toml).
2. Click the pencil icon in the top-right corner.
3. Find the KV configuration:

```toml
[[kv_namespaces]]
binding = "TAFENG_KV"
id = "replace-with-production-kv-id"
preview_id = "replace-with-preview-kv-id"
```

4. Replace the placeholder values with the KV IDs copied from Cloudflare:

```toml
[[kv_namespaces]]
binding = "TAFENG_KV"
id = "your-production-kv-id"
preview_id = "your-preview-kv-id"
```

5. Confirm the R2 bucket names:

```toml
[[r2_buckets]]
binding = "TAFENG_FILES"
bucket_name = "tafeng-files"
preview_bucket_name = "tafeng-files-preview"
```

6. Click `Commit changes` to save the edit to your fork.

### 5. Connect the GitHub Repository in Cloudflare

Cloudflare Workers can build and deploy from a GitHub repository. The exact Dashboard labels may change over time, but the flow is usually:

1. Open the Cloudflare Dashboard.
2. Go to `Workers & Pages`.
3. Click `Create` or `Create application`.
4. Choose a Git repository import option, usually named `Import a repository`, `Connect to Git`, or similar.
5. Select `GitHub`.
6. If this is your first time connecting GitHub, authorize the Cloudflare GitHub App.
7. During authorization, select your forked `tafeng` repository.
8. Back in Cloudflare, choose the repository and the branch to deploy, usually `main`.

If the page asks for a project type, choose `Workers`, not a plain static Pages project.

### 6. Fill in Build Settings

Cloudflare may automatically read [wrangler.toml](./wrangler.toml). If it asks you to fill fields manually, use:

```text
Project name: tafeng
Production branch: main
Root directory: /
Build command: npm run build
Deploy command: npm run worker:deploy
Node.js version: 20
```

If the page asks for a static assets or output directory, use:

```text
dist/client
```

If the page only provides one command field, use:

```bash
npm run build && npm run worker:deploy
```

Cloudflare's Workers Git integration UI can vary slightly between releases. The key idea is: run `npm run build` to generate `dist/client`, then let Wrangler deploy the Worker using `wrangler.toml`.

### 7. Set Environment Variables and Secrets

The first deployment may still work with the default password `tafeng`, but production must use your own admin password.

After the Worker is created:

1. Open `Workers & Pages` in Cloudflare Dashboard.
2. Open the `tafeng` Worker.
3. Go to `Settings`.
4. Open `Variables and Secrets`.
5. Add a Secret:

```text
Name: ADMIN_PASSWORD
Value: your-strong-admin-password
```

Optionally add:

```text
Name: SESSION_SECRET
Value: a long random string
```

If the Dashboard separates `Production` and `Preview` environments, set `ADMIN_PASSWORD` at least for `Production`.

### 8. Redeploy

After adding Secrets, redeploy once:

1. Open the Worker's `Deployments` or `Builds` page.
2. Find the latest deployment.
3. Click `Retry deployment`, `Redeploy`, or a similar button.

You can also make a tiny README edit in GitHub and commit it to trigger a new Cloudflare build.

### 9. Open Tafeng

After deployment, Cloudflare will provide a `workers.dev` URL, for example:

```text
https://tafeng.your-subdomain.workers.dev
```

Open it:

1. Enter the password configured in `ADMIN_PASSWORD`.
2. Enter the console.
3. Add VPS connection profiles from the left panel.
4. The current version is still in development bridge mode. Real SSH/SFTP needs to be implemented in [worker/sshBridge.ts](./worker/sshBridge.ts) using Worker TCP Socket.

### 10. Configure a Custom Domain

To use your own domain:

1. Open the Cloudflare Dashboard.
2. Go to `Workers & Pages`.
3. Open the `tafeng` Worker.
4. Go to `Settings`.
5. Open `Domains & Routes`.
6. Add a custom domain or route.

Example:

```text
ssh.example.com
```

Save it and wait for DNS and certificate provisioning to complete.

## Local Development

Install dependencies:

```bash
npm install
```

Start the Vite frontend dev server:

```bash
npm run dev
```

Default frontend URL:

```text
http://localhost:5173/
```

Build the frontend and start the local Worker:

```bash
npm run build
npm run worker:dev
```

Default Worker URL:

```text
http://localhost:8787/
```

The local development admin password is:

```text
tafeng
```

Do not use the default password in production. Configure the `ADMIN_PASSWORD` secret instead.

## Cloudflare Resources

Tafeng needs two Cloudflare resources:

- KV Namespace: stores settings, sessions, VPS connection profiles, command history index, and command history entries.
- R2 Bucket: stores uploaded files or large-file transfer staging data.

### 1. Log In to Cloudflare

```bash
npx wrangler login
```

For headless server deployments, use a Cloudflare API Token:

```bash
export CLOUDFLARE_API_TOKEN="your-cloudflare-api-token"
```

The token needs permissions for Workers, KV, and R2.

### 2. Create KV Namespaces

Create the production KV namespace:

```bash
npx wrangler kv namespace create TAFENG_KV
```

Create the preview KV namespace:

```bash
npx wrangler kv namespace create TAFENG_KV --preview
```

Wrangler will output something similar to:

```toml
[[kv_namespaces]]
binding = "TAFENG_KV"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
preview_id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy"
```

Copy `id` and `preview_id` into [wrangler.toml](./wrangler.toml):

```toml
[[kv_namespaces]]
binding = "TAFENG_KV"
id = "your-production-kv-id"
preview_id = "your-preview-kv-id"
```

### 3. Create R2 Buckets

Create the production bucket:

```bash
npx wrangler r2 bucket create tafeng-files
```

Create the preview bucket:

```bash
npx wrangler r2 bucket create tafeng-files-preview
```

Make sure [wrangler.toml](./wrangler.toml) matches the bucket names:

```toml
[[r2_buckets]]
binding = "TAFENG_FILES"
bucket_name = "tafeng-files"
preview_bucket_name = "tafeng-files-preview"
```

### 4. Set the Admin Password

Set a production admin password:

```bash
npx wrangler secret put ADMIN_PASSWORD
```

Enter your password when prompted.

Optional: set a session secret. The current code reserves `SESSION_SECRET` for future signed sessions or encrypted storage:

```bash
npx wrangler secret put SESSION_SECRET
```

## Build and Deploy

### 1. Type Check

```bash
npm run typecheck
```

### 2. Build the Frontend

```bash
npm run build
```

The frontend output is written to:

```text
dist/client/
```

### 3. Deploy the Worker

```bash
npm run worker:deploy
```

After deployment, Wrangler will print a Worker URL, for example:

```text
https://tafeng.your-subdomain.workers.dev
```

Open the URL and log in with the password configured through `ADMIN_PASSWORD`.

## Custom Domain

You can bind a custom domain in the Cloudflare Dashboard:

1. Open the Cloudflare Dashboard.
2. Go to Workers & Pages.
3. Select the `tafeng` Worker.
4. Open Settings.
5. Add a custom domain or route under Domains & Routes.

## Real SSH/SFTP Integration Path

The Worker-side adapter lives here:

```text
worker/sshBridge.ts
```

For real SSH integration:

1. Receive browser terminal input through the Worker WebSocket.
2. Use `connect()` from `cloudflare:sockets` inside the Worker to open an outbound TCP connection to the VPS `host:22`.
3. Implement or integrate SSH protocol handling inside the Worker.
4. Pipe browser WebSocket data and SSH TCP Socket data in both directions.
5. Implement SFTP in the same Worker adapter layer. Use R2 as temporary object storage when needed.

Notes:

- Workers can create outbound TCP connections, but they do not behave like traditional servers listening on arbitrary TCP ports.
- Real SSH, SFTP, large files, and long-lived sessions must account for Worker limits, timeouts, memory, and concurrent open connections.
- For 10 GB files, write to R2 first and process the transfer in chunks or tasks. Do not read the whole file into memory.

## Two-Factor Authentication

The UI already includes a two-factor authentication switch and the login flow placeholder. In development mode, the placeholder code is:

```text
000000
```

For production, integrate a standard TOTP flow:

- Generate a TOTP secret for the administrator.
- Bind it through a QR code in an Authenticator app.
- Verify TOTP in the Worker login logic.
- Store the TOTP secret encrypted in KV or as a Worker Secret.

Related files:

- [worker/auth.ts](./worker/auth.ts)
- [src/components/SettingsPanel.tsx](./src/components/SettingsPanel.tsx)

## File Upload and Download

Upload endpoint:

```text
POST /api/files/upload
```

The Worker checks `Content-Length` and returns `413` for files larger than 10 GB. Uploaded content is written to the R2 bucket:

```text
TAFENG_FILES
```

Recommended flow after real SFTP integration:

1. Browser uploads to the Worker.
2. Worker streams the large file into a temporary R2 object.
3. The Worker SSH/SFTP adapter reads from R2 and uploads to the VPS.
4. Temporary R2 objects are deleted after completion.

## Command History

Tafeng stores executed commands globally across all VPS connections, up to 100000 entries.

Endpoints:

```text
GET /api/command-history?limit=300&offset=0
DELETE /api/command-history
```

Storage design:

- KV stores a global index.
- Each command is stored as a separate KV entry.
- When the history exceeds 100000 entries, the oldest entries are deleted automatically.

## Multilingual UI

Currently supported:

- Chinese
- English

Frontend translations are stored in:

```text
src/lib/i18n.ts
```

To add a new language:

1. Extend the `Language` type in `shared/types.ts`.
2. Add a dictionary in `src/lib/i18n.ts`.
3. Add an option in the language selector inside `SettingsPanel`.

## Security Checklist

Before production deployment:

- Set a strong admin password with `npx wrangler secret put ADMIN_PASSWORD`.
- Integrate real TOTP two-factor authentication.
- Do not store SSH passwords or private keys in plaintext long term.
- Add encryption for VPS credentials stored in KV.
- Restrict Worker access or add additional access control if needed.
- Periodically clear unnecessary command history.
- Add cleanup rules for temporary R2 upload objects.
- Add connection timeouts, command auditing, and robust error handling for real SSH integration.

## Common Commands

```bash
# Install dependencies
npm install

# Start frontend dev server
npm run dev

# Type check
npm run typecheck

# Build frontend
npm run build

# Run Worker locally
npm run worker:dev

# Deploy Worker
npm run worker:deploy
```

## FAQ

### 1. I cannot log in.

Make sure the production admin password is configured:

```bash
npx wrangler secret put ADMIN_PASSWORD
```

When no Secret is configured during local development, the default password is `tafeng`.

### 2. Wrangler says the KV id is invalid.

Make sure you have run:

```bash
npx wrangler kv namespace create TAFENG_KV
npx wrangler kv namespace create TAFENG_KV --preview
```

Then copy the generated `id` and `preview_id` into [wrangler.toml](./wrangler.toml).

### 3. File upload fails.

Make sure the R2 buckets exist and that `bucket_name` and `preview_bucket_name` in [wrangler.toml](./wrangler.toml) are correct.

### 4. The terminal does not connect to a real VPS yet.

This is expected for the current prototype. Real SSH/SFTP needs to be implemented in [worker/sshBridge.ts](./worker/sshBridge.ts) using Worker TCP Socket and SSH protocol logic.

### 5. Is command history isolated by VPS?

No. The current design uses global command history. Commands from all VPS connections go into one shared list, up to 100000 entries.

## License

This project is licensed under the MIT License. You may use, modify, distribute, and use it commercially, as long as the copyright and license notices are preserved. See [LICENSE](./LICENSE).
