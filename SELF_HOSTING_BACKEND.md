<!--
SPDX-FileCopyrightText: NONE

SPDX-License-Identifier: CC0-1.0
-->

# Self-hosting the backend (real cross-device save sync)

Offline mode (`VITE_BYPASS_LOGIN=1`) stores saves only in one browser. For real
accounts and automatic cloud sync across devices, run your own copy of the
PokéRogue backend, [`rogueserver`](https://github.com/pagefaultgames/rogueserver),
and point this frontend at it.

> ⚠️ The backend must run on an **always-on host** (a small VPS or a PaaS), not on a
> laptop or this ephemeral environment. It also needs a **public HTTPS URL** — a
> Netlify frontend is served over HTTPS, and browsers block calls from an HTTPS page
> to an `http://` backend (mixed content).

## Architecture

```
iPhone / browser ──HTTPS──> Netlify (this frontend)
                              │  fetch(VITE_SERVER_URL)
                              ▼
                         HTTPS (Caddy/TLS)
                              │  reverse proxy
                              ▼
                     rogueserver (:8001 API)  ──>  MariaDB
```

## Step 1 — Get an always-on host

Anything that can run Docker works: a cheap VPS (Hetzner, DigitalOcean, Vultr, etc.)
or a PaaS that runs containers (Railway, Render, Fly.io). You also need a **domain or
subdomain** you control (e.g. `api.yourgame.com`) so you can serve the backend over
HTTPS.

## Step 2 — Run rogueserver + MariaDB

```bash
git clone https://github.com/pagefaultgames/rogueserver.git
cd rogueserver
# Dev compose brings up MariaDB + the Go server and auto-creates the schema.
docker compose -f docker-compose.Development.yml up -d
```

The API listens on **:8001** (this is the URL the frontend talks to). Configure DB
credentials via the documented flags/vars (`dbuser`, `dbpass`, plus db host/name in
`rogueserver.go`). Use strong credentials for a public deployment.

## Step 3 — Put it behind HTTPS

Point your domain at the host and terminate TLS with a reverse proxy. Example
[Caddy](https://caddyserver.com) config:

```
api.yourgame.com {
    reverse_proxy localhost:8001
}
```

Caddy fetches a free certificate automatically. Your backend API is now
`https://api.yourgame.com`.

## Step 4 — Allow your frontend's origin (CORS)

rogueserver must accept requests from your Netlify origin
(`https://<your-site>.netlify.app`). Check rogueserver's CORS / allowed-origins
configuration and add that origin. (Because the browser calls the backend
cross-origin, the backend — not the browser — has to permit it.)

## Step 5 — Point this frontend at your backend

Switch the Netlify build out of offline mode and at your API. In `netlify.toml`,
replace the `VITE_BYPASS_LOGIN=1` build command with:

```toml
[build]
  command = "VITE_BYPASS_LOGIN=0 VITE_SERVER_URL=https://api.yourgame.com pnpm build && cp -r assets/. dist/ && mkdir -p dist/locales && cp -r locales/. dist/locales/"
```

Redeploy. The login screen now talks to your server, and saves sync to your
database across every device that logs into the same account.

## Notes

- **OAuth (Discord/Google)** login depends on provider redirect URIs and extra setup;
  username/password is the straightforward path for a self-host.
- **Migrating existing progress:** create an account on your server, then use in-game
  **Manage Data → Import** to load a `.prsv` you exported (e.g. from your offline
  build or from desktop `pokerogue.net`). From then on it syncs automatically.
- Keep the backend updated alongside the frontend version to avoid save-format drift.
