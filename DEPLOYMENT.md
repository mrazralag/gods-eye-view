# Running and Deploying This Checkout

Notes from a verified build of this branch. Everything below was actually
executed against this tree — no changes to application code were needed.

## 1. Toolchain

`package.json` sets `engines: ">=24.14.0 <25 || >=26 <27"`. Node 22 will not do;
the repo's CI matrix is 24.14.0 and 26.x.

```bash
nvm install        # reads .nvmrc → 24.21.0
nvm use
node -v            # v24.21.0
```

## 2. Build

```bash
npm ci
npm run doctor     # toolchain + provider readiness, prints no secrets
npm run dev        # http://localhost:4173  ← the way this app is meant to run
```

Verified on this branch with Node 24.21.0 / npm 11.19.0:

| Gate | Command | Result |
|---|---|---|
| Dependencies | `npm ci` | 126 packages, 0 vulnerabilities, ~13s |
| Setup policy | `npm run doctor` | Ready, keyless |
| Formatting | `npm run format:check` | 71 adopted files clean |
| Package boundaries | `npm run check:boundaries` | clean |
| Unit tests | `npm test` | 2966 passed, 0 failed (~110s) |
| Production bundle | `npm run build` | built in ~5s, `dist/` ≈ 28 MB |

A headless Chromium load of the built bundle reaches `Initializing systems...`
with WebGL 2.0, `window.Cesium` defined, and the HUD chrome rendered.

npm 11 blocks lifecycle scripts by default. `esbuild` still resolves through its
platform package, so the build is unaffected; only `puppeteer`'s browser
download is skipped, which matters solely for the `scripts/qa-*.mjs` harnesses.
CI sets `PUPPETEER_SKIP_DOWNLOAD=1` for the same reason.

## 3. The thing to understand before deploying

**The live-data proxies are Vite dev-server middleware, not a standalone
service.** `server/providers/*` registers them through Vite plugin hooks, and
most register `configureServer` only. So `dist/` on a static host is a
materially different application:

| Surface | `npm run dev` | `npm run preview` | Static `dist/` |
|---|---|---|---|
| Globe, HUD, basemap, bundled datasets | ✅ | ✅ | ✅ |
| Flights (OpenSky + adsb.lol fallback, adsbdb enrichment) | ✅ | ❌ | ❌ |
| Satellites (CelesTrak) | ✅ | ❌ | ❌ |
| Fires (FIRMS), bike share (GBFS), traffic (TomTom) | ✅ | ❌ | ❌ |
| CCTV, Overpass, terrain heights | ✅ | ❌ | ❌ |
| Vessels (AISStream), radio, launches, OpenAI voice, Google places, weather effects, regional brief, track backfill | ✅ | ✅ | ❌ |
| POWER UP key panel (`/api/setup/*`) | ✅ | ❌ (by design) | ❌ |

Confirmed by probing both servers: `/api/setup/status` and `/api/celestrak`
return the SPA HTML fallback under preview but real JSON under dev;
`/api/launches` and `/api/radio` are real handlers in both.

The key panel is deliberately dev-only — it writes `.env` and restarts the
server, so it never ships to a hosted build. Upstream states the project is
"a local-first client… not a hardened production service," and there is no
supported production deployment path.

**Practical consequence:** to run the real product, run the dev server. Treat
`npm run build` as the CI gate it is in `.github/workflows/ci.yml`, not as a
deployment artifact — unless you accept the globe-only subset, or port the
provider middleware into a real HTTP service (Express/Fastify) sitting in front
of `dist/`.

## 4. Keys

None are required to start: Esri World Imagery + keyless terrain, OpenSky
anonymous flights, USGS, CelesTrak, adsb.lol, CCTV, Radio Browser, GBFS and
Launch Library 2 all work with no signup.

Add keys through **POWER UP → Provider Settings** in the running app (writes
repo-root `.env`, chmod 600), or write `.env` yourself from `.env.example`.

| Variable | Unlocks | Cost |
|---|---|---|
| `CESIUM_ION_TOKEN` | Photorealistic 3D + world terrain | Free tier, personal/non-commercial |
| `GOOGLE_MAPS_API_KEY` | Direct Google 3D tiles + place search | Metered, billing required |
| `GOOGLE_MAPS_SERVER_API_KEY` | Server-side places / Street View | Metered |
| `OPENAI_API_KEY` | Voice control + AI HUD summary | Metered; app caps a session at $5 |
| `AISSTREAM_API_KEY` | Live vessels | Free signup |
| `FIRMS_MAP_KEY` | Live active fires | Free |
| `TOMTOM_API_KEY` | Live traffic flow | Free tier |
| `OPENSKY_CLIENT_ID` / `_SECRET` | Higher flight-poll allowance | Free |
| `LL2_API_TOKEN` | Higher launch-data allowance | Free |

`GOOGLE_MAPS_API_KEY` and `CESIUM_ION_TOKEN` are injected into the browser
bundle at build time (`build/vite.js` `define`). They end up in `dist/` in
plain text. Restrict both at the provider (HTTP referrer + API restriction for
Google, URL-restricted `assets:read` for ion) before hosting anything. Every
other key stays server-side and is only ever reachable through the proxies.

## 5. If you expose it beyond localhost

The server binds to localhost on both launch paths. `--host 0.0.0.0` turns your
machine into an open key broker for anyone on the network — your OpenAI and
Google credentials, spent by strangers. Before doing it: set provider quotas and
billing alerts (app throttles are not billing caps), set
`GEV_RATELIMIT_OPENAI_PER_MIN` and `GEV_RATELIMIT_GOOGLE_PER_MIN`, and read
`SECURITY.md`. Provider Settings disables itself automatically when the server
is shared.
