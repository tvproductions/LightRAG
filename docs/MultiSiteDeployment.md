# Single-Server Multi-Site Deployment

This document explains how to run multiple isolated LightRAG instances
behind one host using a reverse proxy (nginx, Traefik, Kubernetes Ingress,
…), with **one shared WebUI build** reused by every instance.

> Looking for the basic single-instance Docker setup? See
> [DockerDeployment.md](./DockerDeployment.md). For frontend build
> mechanics in general, see [FrontendBuildGuide.md](./FrontendBuildGuide.md).

---

## TL;DR

- Set `LIGHTRAG_API_PREFIX` and `LIGHTRAG_WEBUI_PATH` per-instance, on the
  **backend only**.
- Build the WebUI **once**. The same artifacts work under any reverse-proxy
  prefix.
- Point your reverse proxy at each backend, stripping the site prefix
  before forwarding.

```bash
# One image, two containers, two prefixes — no rebuild.
docker run -e LIGHTRAG_API_PREFIX=/site01 -p 9621:9621 lightrag:latest
docker run -e LIGHTRAG_API_PREFIX=/site02 -p 9622:9621 lightrag:latest
```

---

## Why "build once, deploy many"

Earlier versions of LightRAG baked the site prefix into the JavaScript
bundle at build time (via `VITE_API_PREFIX` / `VITE_WEBUI_PREFIX`). Every
site that used a different prefix needed its own WebUI build, and reusing
a single Docker image across sites required a rebuild step at deploy time.

Since the runtime-config-injection refactor:

- **Asset URLs** in `index.html` are emitted as relative paths
  (`./assets/index-abc.js`). The browser resolves them against the current
  document URL, so they work under any mount point.
- **API base URL** and **in-app links** read their prefix from
  `window.__LIGHTRAG_CONFIG__`, which the FastAPI server injects into
  `index.html` on each response based on its own
  `LIGHTRAG_API_PREFIX` / `LIGHTRAG_WEBUI_PATH`.

The result: a single `lightrag/api/webui/` directory (or Docker image) is
reusable across any number of sites with no per-site build artifact.

---

## How runtime prefix injection works

Each request for `index.html` goes through `SmartStaticFiles` in
`lightrag/api/lightrag_server.py`, which:

1. Reads the static `index.html` produced by `bun run build`.
2. Looks for the placeholder comment
   `<!-- __LIGHTRAG_RUNTIME_CONFIG__ -->`.
3. Replaces it with
   `<script>window.__LIGHTRAG_CONFIG__ = {"apiPrefix":"…","webuiPrefix":"…"}</script>`,
   computed from the configured `LIGHTRAG_API_PREFIX` / `LIGHTRAG_WEBUI_PATH`.

Sequence — browser request to a site-prefixed instance:

```
Browser            nginx                  uvicorn            SmartStaticFiles
  │                  │                       │                       │
  │ GET /site01/webui/                       │                       │
  │─────────────────►│                       │                       │
  │                  │ GET /webui/  (strips /site01)                 │
  │                  │──────────────────────►│                       │
  │                  │                       │ get_response("")      │
  │                  │                       │──────────────────────►│
  │                  │                       │                       │ inject
  │                  │                       │                       │ window.__LIGHTRAG_CONFIG__
  │                  │                       │                       │  = { apiPrefix: "/site01",
  │                  │                       │                       │      webuiPrefix: "/site01/webui/" }
  │                  │                       │◄──────────────────────│
  │                  │◄──────────────────────│                       │
  │◄─────────────────│                       │                       │
  │ index.html with injected runtime config                          │
```

The SPA reads the injected config via `src/lib/runtimeConfig.ts` and uses
it for `axios.baseURL`, `fetch()` template strings, the API-docs iframe,
and in-app links.

---

## Two backend variables, that's it

| Variable | Default | Meaning |
| --- | --- | --- |
| `LIGHTRAG_API_PREFIX` | `""` | Reverse-proxy prefix that the upstream proxy strips before forwarding to FastAPI. Passed to FastAPI as `root_path`. |
| `LIGHTRAG_WEBUI_PATH` | `/webui` | In-app mount path for the WebUI **after** the proxy has stripped the API prefix. Leave as `/webui` unless you have a specific reason to relocate it. |

`window.__LIGHTRAG_CONFIG__.webuiPrefix` is computed as
`LIGHTRAG_API_PREFIX + LIGHTRAG_WEBUI_PATH + "/"`. You do **not** set this
yourself.

There are no longer any frontend `VITE_API_PREFIX` / `VITE_WEBUI_PREFIX`
variables. Setting them has no effect (they are ignored by the build).

---

## End-to-end example: two sites behind one nginx

### Instance configuration

`site01.env`:
```bash
HOST=0.0.0.0
PORT=9621
LIGHTRAG_API_PREFIX=/site01
LIGHTRAG_WEBUI_PATH=/webui
WORKING_DIR=/data/site01/storage
INPUT_DIR=/data/site01/inputs
LIGHTRAG_API_KEY=site01-secret
# … LLM / embedding config …
```

`site02.env`:
```bash
HOST=0.0.0.0
PORT=9621
LIGHTRAG_API_PREFIX=/site02
LIGHTRAG_WEBUI_PATH=/webui
WORKING_DIR=/data/site02/storage
INPUT_DIR=/data/site02/inputs
LIGHTRAG_API_KEY=site02-secret
# … LLM / embedding config …
```

### docker-compose.yml (one image, two services)

```yaml
services:
  site01:
    image: ghcr.io/hkuds/lightrag:latest
    env_file: site01.env
    volumes:
      - ./data/site01:/data/site01
    ports:
      - "127.0.0.1:9621:9621"

  site02:
    image: ghcr.io/hkuds/lightrag:latest
    env_file: site02.env
    volumes:
      - ./data/site02:/data/site02
    ports:
      - "127.0.0.1:9622:9621"
```

### nginx config

```nginx
server {
    listen 443 ssl http2;
    server_name host.example.com;

    # site01: strips /site01/ before forwarding
    location /site01/ {
        proxy_pass http://127.0.0.1:9621/;
        proxy_set_header X-Forwarded-Prefix /site01;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # site02: strips /site02/ before forwarding
    location /site02/ {
        proxy_pass http://127.0.0.1:9622/;
        proxy_set_header X-Forwarded-Prefix /site02;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

Browsing `https://host.example.com/site01/webui/` shows site01's WebUI;
`https://host.example.com/site02/webui/` shows site02's. The same Docker
image serves both — no per-site build artifact, no rebuild on prefix
changes.

### What each layer sees

| Layer | site01 GET /webui/ |
| --- | --- |
| Browser address bar | `https://host.example.com/site01/webui/` |
| nginx receives | `/site01/webui/` |
| nginx forwards | `/webui/` |
| FastAPI `root_path` | `/site01` |
| `app.mount` resolves | `/webui/` |
| Injected `apiPrefix` | `/site01` |
| Injected `webuiPrefix` | `/site01/webui/` |
| Asset URLs in HTML | `./assets/index-abc.js` (resolves to `https://host.example.com/site01/webui/assets/index-abc.js`) |

---

## Single-image Docker recipe

The `Dockerfile` builds the WebUI once, with no prefix:

```dockerfile
FROM oven/bun:1 AS webui-build
WORKDIR /src/lightrag_webui
COPY lightrag_webui/package.json lightrag_webui/bun.lock ./
RUN bun install --frozen-lockfile
COPY lightrag_webui/ ./
COPY lightrag/api/webui/.gitkeep /src/lightrag/api/webui/.gitkeep
RUN bun run build

FROM python:3.11-slim
COPY --from=webui-build /src/lightrag/api/webui /app/lightrag/api/webui
# … rest of the image …
```

Run any number of containers from the same image, each with its own
prefix:

```bash
# Plain single-instance, no prefix.
docker run --rm -p 9621:9621 lightrag:latest

# Same image, different prefixes — runtime decides.
docker run --rm -e LIGHTRAG_API_PREFIX=/site01 -p 9621:9621 lightrag:latest
docker run --rm -e LIGHTRAG_API_PREFIX=/site02 -p 9622:9621 lightrag:latest

# Custom in-app mount.
docker run --rm \
  -e LIGHTRAG_API_PREFIX=/team-a \
  -e LIGHTRAG_WEBUI_PATH=/admin-ui \
  -p 9623:9621 \
  lightrag:latest
```

### Kubernetes Ingress equivalent

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lightrag-multisite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: host.example.com
    http:
      paths:
      - path: /site01(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: lightrag-site01
            port: { number: 9621 }
      - path: /site02(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: lightrag-site02
            port: { number: 9621 }
```

Backends still set `LIGHTRAG_API_PREFIX=/site01` / `=/site02`.

---

## Local development with `bun run dev`

The dev server mirrors production injection: it serves `index.html` via
the same `transformIndexHtml` mechanism the FastAPI server uses at request
time, so the SPA reads `window.__LIGHTRAG_CONFIG__` in dev exactly the
way it does in prod. Only **two** environment variables matter:

| Variable | Purpose | Where it lives |
| --- | --- | --- |
| `VITE_BACKEND_URL` | Where the dev server forwards proxied API calls. | `lightrag_webui/.env*` |
| `VITE_DEV_API_PREFIX` | Prefix to **simulate** (matches the production `LIGHTRAG_API_PREFIX`). Empty → no prefix. | `lightrag_webui/.env*` |

`VITE_DEV_WEBUI_PREFIX` is also accepted but only affects the home/logo
`<a href>` link inside the SPA — set it to `${VITE_DEV_API_PREFIX}/webui/`
if you care about that link in dev, otherwise leave it empty.

Three scenarios cover everything you'll hit:

### Scenario 1 — single-instance dev (no prefix, no proxy)

The default. Don't set anything beyond the existing `.env.development`.

```
Browser ──► localhost:5173 (Vite) ──► localhost:9621 (backend, no prefix)
```

```bash
# lightrag_webui/.env.development (already in repo as sample)
VITE_BACKEND_URL=http://localhost:9621
VITE_API_PROXY=true
VITE_API_ENDPOINTS=/api,/documents,/graphs,/graph,/health,/query,/docs,/redoc,/openapi.json,/login,/auth-status,/static
# VITE_DEV_API_PREFIX=          ← leave empty
```

Run:
```bash
lightrag-server                  # in one terminal, no LIGHTRAG_API_PREFIX
cd lightrag_webui && bun run dev # in another; open http://localhost:5173/
```

### Scenario 2 — simulate the production prefix WITHOUT running nginx (recommended)

You want to develop against a prefix-configured backend, but don't want
to install / configure nginx locally just to debug. **The Vite dev
server's built-in proxy plays the role of the reverse proxy.**

```
Browser ──► localhost:5173 (Vite, simulates /site01) ──► localhost:9621 (backend, root_path=/site01)
                                                          (no nginx in this picture)
```

Setup:

```bash
# lightrag_webui/.env.local  (gitignored — your personal dev config)
VITE_BACKEND_URL=http://localhost:9621
VITE_API_PROXY=true
VITE_API_ENDPOINTS=/api,/documents,/graphs,/graph,/health,/query,/docs,/redoc,/openapi.json,/login,/auth-status,/static
VITE_DEV_API_PREFIX=/site01
VITE_DEV_WEBUI_PREFIX=/site01/webui/
```

```bash
# Terminal 1 — start the backend with the matching prefix
LIGHTRAG_API_PREFIX=/site01 lightrag-server

# Terminal 2 — start the dev server
cd lightrag_webui && bun run dev
```

Then open **`http://localhost:5173/`** (root, NOT `/site01/`). Vite serves
the SPA at `/`; the SPA generates prefixed API URLs at runtime.

What happens:

- `vite.config.ts` injects `window.__LIGHTRAG_CONFIG__ = { apiPrefix: "/site01", … }` into the dev `index.html`.
- The SPA emits requests like `fetch("/site01/documents/foo")`.
- `server.proxy` matches `/site01/documents` and forwards verbatim to `http://localhost:9621/site01/documents/foo`.
- The backend (`root_path=/site01`) accepts the prefixed path and serves it.

HMR continues to work unchanged. **No nginx, no Docker — just two
processes on localhost.**

### Scenario 3 — local dev frontend against a real (remote) backend

You want to iterate on the WebUI on your laptop while hitting a backend
that's already in production behind nginx — typically because the real
backend has data / configs that are painful to reproduce locally. The
WebUI is purely local (HMR on your laptop); only API traffic crosses the
network.

```
Local machine                                Remote production host
─────────────────                            ────────────────────────
Browser ──► localhost:5173 (Vite + HMR)
                │
                │  Vite proxy forwards /site01/* verbatim
                ▼
              ─── network ───►  nginx ──strips /site01/──► lightrag-server
                                                          (backend; may
                                                           or may not have
                                                           LIGHTRAG_API_PREFIX
                                                           set — both work)
```

The key insight: the production nginx is **already** doing the prefix
strip. Vite's only job is to forward prefixed paths to nginx unchanged,
and nginx behaves exactly as it would for a real production browser
request.

**Setup:**

1. **Production nginx + backend:** unchanged. Whatever your real deploy
   already runs.

2. **Dev server (`.env.local`):**
   ```bash
   # Point at the production reverse proxy — NOT the backend port.
   VITE_BACKEND_URL=https://prod.example.com         # or http://10.0.0.5
   VITE_API_PROXY=true
   VITE_API_ENDPOINTS=/api,/documents,/graphs,/graph,/health,/query,/docs,/redoc,/openapi.json,/login,/auth-status,/static
   VITE_DEV_API_PREFIX=/site01
   VITE_DEV_WEBUI_PREFIX=/site01/webui/
   ```

3. Run `bun run dev` and open **`http://localhost:5173/`**.

What happens for an API call:

- SPA fetches `/site01/documents/foo` (because `apiPrefix=/site01` was
  injected into `window.__LIGHTRAG_CONFIG__`).
- Vite's `server.proxy` matches `/site01/documents` and forwards verbatim
  to `https://prod.example.com/site01/documents/foo`.
  (`changeOrigin: true` is already set, so the `Host` header is rewritten
  to the upstream — required for SNI / virtual hosts.)
- Production nginx matches `/site01/`, strips it, forwards
  `/documents/foo` to the backend.
- Backend serves it. `LIGHTRAG_API_PREFIX` on the backend can be set or
  unset; FastAPI's `root_path` accepts both prefixed and natural forms
  either way.

HMR is purely local — the browser only talks to localhost:5173 for SPA
assets. No nginx involvement, no special WebSocket-upgrade config to
worry about.

#### Why `VITE_BACKEND_URL` does **not** include `/site01`

Vite forwards the request path **verbatim** (no rewrite). The browser
already emits `/site01/documents/foo`, so the URL Vite sends upstream is
`${VITE_BACKEND_URL}/site01/documents/foo`. If you set
`VITE_BACKEND_URL=https://prod.example.com/site01` you'd get
`https://prod.example.com/site01/site01/documents/foo` — a duplicated
prefix that nginx and the backend both reject.

Same logic applies to Scenario 2 (`VITE_BACKEND_URL=http://localhost:9621`,
no prefix): Vite forwards `/site01/documents/foo` unchanged to the
backend, and FastAPI's `root_path=/site01` matches the prefixed form
natively.

#### Common pitfalls

- **HTTPS upstream + self-signed cert**: Vite's proxy will reject by
  default. Set `proxy: { ..., secure: false }` in `vite.config.ts` to
  skip cert validation when targeting a staging proxy with a non-public
  cert.
- **Auth required**: if the production backend requires `LIGHTRAG_API_KEY`,
  log in via the dev SPA exactly as you would in prod — the auth token
  flows through the proxy unchanged.
- **CORS errors**: shouldn't happen because the browser sees same-origin
  requests to localhost:5173. If they appear, check that
  `changeOrigin: true` is in effect (it is, by default in
  `vite.config.ts`).

### Quick decision matrix

| You want to… | `VITE_BACKEND_URL` | `VITE_DEV_API_PREFIX` | What's in front of the backend | Open in browser |
| --- | --- | --- | --- | --- |
| Default single-instance dev | `http://localhost:9621` | unset | local backend, no proxy | `http://localhost:5173/` |
| Reproduce a multi-site bug locally | `http://localhost:9621` | `/site01` | local backend with `LIGHTRAG_API_PREFIX=/site01`, no nginx | `http://localhost:5173/` |
| Hit a real (remote) backend through its production nginx | `https://prod.example.com` | `/site01` | production nginx already strips `/site01/` | `http://localhost:5173/` |

In every case the browser only talks to local Vite at `localhost:5173`;
the column on the right is the same. Where the API traffic ultimately
lands is what differs.

---

## Migration notes

If you were on the previous build-time-prefix model:

- **Stop setting `VITE_API_PREFIX` and `VITE_WEBUI_PREFIX`.** They are
  ignored by the new build. Remove them from your CI / build scripts.
- **Drop per-site Docker images.** A single image works for every prefix.
  CI no longer needs a "build once per site" matrix.
- **No more "prefix mismatch" warnings at startup.** The
  `check_webui_build_prefix` function and its banner have been removed —
  there is nothing to mismatch.
- **The `lightrag_webui/index.html` template now contains the placeholder
  comment `<!-- __LIGHTRAG_RUNTIME_CONFIG__ -->`.** If you fork the
  template, keep that line in `<head>` or the runtime config will not be
  injected (the SPA falls back to no-prefix defaults).

---

## Troubleshooting

### Asset URLs 404 when accessing the WebUI

The base URL must end with `/`. Accessing `/site01/webui` (no trailing
slash) makes the browser resolve `./assets/foo.js` against `/site01/`,
which 404s. The server already redirects the no-slash form to the
slash form; verify the redirect is reaching nginx (check
`X-Forwarded-Prefix` and that nginx uses `proxy_pass http://…/` with the
trailing slash).

### `apiPrefix` is empty in `window.__LIGHTRAG_CONFIG__` after deploy

View the page source. If you see the literal placeholder
`<!-- __LIGHTRAG_RUNTIME_CONFIG__ -->` instead of an injected
`<script>` tag, the request did not go through `SmartStaticFiles` —
double-check that `lightrag/api/webui/index.html` exists in the running
container and that the WebUI mount succeeded (the server logs
`WebUI assets mounted at <path>` at startup).

### `bun run dev` proxy returns 404 with `VITE_DEV_API_PREFIX` set

Confirm the backend is also running with the matching
`LIGHTRAG_API_PREFIX`. The dev proxy forwards prefixed paths verbatim;
if the backend has no prefix configured, it does not register routes
under that path.

### I want to disable the WebUI entirely

Don't build the frontend — `lightrag/api/webui/index.html` will not exist
and the server will skip the WebUI mount, redirecting `/` and the
WebUI path to `/docs` instead. The runtime-config injection is purely
opt-in via the existence of the build artifact.
