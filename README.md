# pizza-node — Cloudflare build

The Express port of the pizza order system, rebuilt as one Cloudflare Worker with static assets and one D1 database.
No Node server, no socket.io, no paid plan.

Branch `recreate`. The `main` branch keeps the original Express tree.

## Live

https://pizza.ennio.vip

This worker replaces the earlier `pizza` worker on the same domain. The D1 database `pizza` is shared, so the menu carries over.

## Architecture

| Path | Handled by | Cost |
| --- | --- | --- |
| `/`, `/sites/**`, `/assets/**`, `/libs/**` | Static assets on the edge cache | 0 Worker requests |
| `GET /api/menu` | Worker: menu lists from D1 | 1 request |
| `POST`/`DELETE /api/menu` | Worker: add or remove one item (admin) | 1 request |
| `GET /api/session` | Worker: active session or `null` | 1 request |
| `POST`/`DELETE /api/session` | Worker: start or end a session (admin) | 1 request |
| `GET /api/orders` | Worker: orders of the session (admin) | 1 request |
| `POST /api/orders` | Worker: place an order | 1 request |

`assets.run_worker_first` is set to `/api/*` only. Static files never start the Worker.

## What replaced what

| Old (main) | New |
| --- | --- |
| `index.js` (Express + cors + body-parser) | `worker/index.js` |
| `POST /api/wtf` — `fs.writeFileSync("public/" + path, ...)`, any path | `/api/menu`, `/api/session`, `/api/orders`, typed and checked |
| `assets/json/ingredients.json` and `options.json` | D1 `menu_items` (13 ingredients, 2 options) |
| socket.io to `myst-socket.glitch.me` (answers 410) | polling `GET /api/session` and `GET /api/orders`, 3 s, ETag |
| session state in the socket server | D1 `sessions` |
| orders pushed over the socket | D1 `orders` |
| password `master` compared in the browser | `x-admin-key` header checked by the Worker |
| `node_modules/` committed (1,185 files) | not tracked |

The admin key is `master` by default. Change `ADMIN_PASSWORD` in `wrangler.jsonc`.

## Kept from this repo

- The menu includes **Brokkoli**.
- An order without a note shows no empty comment row (`pizzaUtils.js`).
- `?v=1` cache busting on the CSS and JS links.

## Fixed while porting

The Options and Ingredients pages appended their list from a network call after `afterLoadStandard` had already run `fadeIn()`. The rows stayed at `display: none`, so the admin saw only the Add/Remove links. The lists now fade in when the data arrives.

## Cost: zero

| Product | Free allowance | What this site uses |
| --- | --- | --- |
| Workers Free | 100,000 requests/day, 10 ms CPU | static files use 0 requests |
| D1 Free | 5M rows read/day, 100k rows written/day, 5 GB | 1 read per poll, 2 writes per order |
| Static assets | 20,000 files, 25 MiB each | 29 files |
| Custom domain | included with the zone | `pizza.ennio.vip` |

## Local development

```bash
npm install          # wrangler 4.30.0; use Node 20+
npm run db:local
npm run dev          # http://127.0.0.1:8787
```

## Deploy

```bash
npm run deploy
```

`wrangler.jsonc` holds `routes: [{ pattern: "pizza.ennio.vip", custom_domain: true }]`, so the deploy creates the DNS record and the certificate.

## Known gaps

- **The admin key is a shared password in a header.** It is not a login session.
- **`/api/orders` is public while a session runs.** Add a Cloudflare WAF rate limit if it gets abused.
- **`adminKey` lives in a page variable.** A reload sends the admin back to the password page.
