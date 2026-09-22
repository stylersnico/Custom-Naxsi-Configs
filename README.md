# Custom Naxsi Configs

My own Naxsi WAF configurations for my self-hosted infrastructure.

The goal of this repository is to provide a hardened reverse-proxy in front of every self-hosted app I run, with support for:

* Naxsi WAF (core rules + per-app hand-written/community whitelists)
* Per-vhost connection & request rate limiting
* Per-vhost error/WAF logging (no `access_log`)
* A shared, locally-served Naxsi "request denied" page
* Global security headers (CSP, HSTS, X-Frame-Options, Permissions-Policy, ...) that always win over whatever the backend sends, via `proxy_hide_header`

Runs on **nginx-full** (FreeBSD 15) with the `naxsi` and `headers_more` dynamic modules.

--------

## Structure

```
Reverse-NGINX/
├── nginx.conf              # main config: modules, TLS, global headers, rate/conn zones
├── sites-enabled/          # one vhost per app
├── naxsi/                  # naxsi_core.rules + per-app whitelists
└── errors/                 # shared naxsi_denied.html
```

--------

## Protected applications

Each app got naxsi in one of two modes:

* **Full scan** – naxsi runs on the whole app, whitelist entries added only after a confirmed false positive in `naxsi_error.log`.
* **Login only** – naxsi runs solely on the authentication endpoint(s); everything past a valid session is bypassed entirely. Used once an app's authenticated surface kept passing structured/serialized data through nearly every field, making field-by-field whitelisting not worth chasing (CheckMK is the case that established the pattern: 5 fixes across 4 endpoints before flipping).

| App | Mode | Notes |
|---|---|---|
| **WordPress** | Full scan | Upstream `wordpress.rules`/`wordpress-block.rules` + local extras (select2/imgareaselect assets, plugin meta-box brackets). `/wp-admin/load-{styles,scripts}.php` bypassed (unenumerable plugin handle list). |
| **Nextcloud** | Full scan | WebDAV (`/remote.php/dav`, `/public.php/webdav`) bypassed - raw XML bodies. Cookie double-encoding + OCS unknown-content-type fixed. |
| **Umami** | Full scan | CORS added on `/api/send` for cross-origin tracking. `title`/`url` fields whitelisted (arbitrary visitor-submitted data). |
| **Gitea** | Full scan | Git smart-HTTP and `/api/` bypassed. Fixed the `/compare/v1...v2` traversal false positive (git's own `...` syntax), login `redirect_to`. |
| **Wiki.js** | Hybrid | `/graphql` (the whole app API - login, page edits) bypassed entirely after 3 fields needed full whitelisting in a row. Static assets still scanned. |
| **Static site** | Full scan | GET/HEAD-only. |
| **CheckMK** | Login only | `login.py` + `user_login_two_factor.py`. |
| **Grafana** | Login only | `/login`. Embeddable in Home Assistant via a scoped CSP `frame-ancestors`. |
| **Home Assistant** | Login only | `/auth/`. `/api/websocket` untouched. |
| **gitea-mirror** | Login only | `/login` + `/api/auth/` (Better Auth). |
| **Passbolt** | Login only | `/auth/login.json` + `/auth/verify.json` (GPG challenge-response). |
| **Jellyfin** | Login only | `POST /Users/AuthenticateByName`. Media streaming untouched. |

No community-maintained naxsi ruleset exists for any of these apps except WordPress (`nbs-system/naxsi-rules`) - every other whitelist is hand-written from real traffic.

--------

## Deployment

> :warning: **Naxsi log directories aren't created automatically.** Before reloading nginx with a new vhost, create its log folder or nginx will refuse to start:
```bash
mkdir -p /var/log/nginx/<vhost>
chown www:www /var/log/nginx/<vhost>
```

Then always test before reloading:
```bash
nginx -t && service nginx reload
```

--------

## Known limitations

* Nextcloud's own admin security panel may still warn about HSTS - that check hits a separate local web server on the Nextcloud host itself, outside this repo.
* Naxsi has no community-validated ruleset for anything but WordPress; every other app's whitelist is scoped to what's actually been seen in production traffic, not a security guarantee against everything.