## Follow Me on Social Media

Stay connected and follow my work on social media:

[![Follow me](https://img.shields.io/badge/Follow%20me-ffffff?logo=x&logoColor=000000)](https://x.com/FunkyxBeatz)

[![Follow Web Frens](https://img.shields.io/badge/Follow%20Web%20Frens-ffffff?logo=x&logoColor=000000)](https://x.com/WebFrens_)


[![Discord](https://img.shields.io/discord/1330332570847547433?label=Join%20the%20Community&logo=discord&logoColor=5865F2&color=5865F2)](https://discord.gg/gVEEv8Yswu)

# Self-Hosted Umami Analytics with Cloudflare Zero Trust

This repository documents the full setup of **Umami Analytics** behind **Cloudflare Tunnels** and **Cloudflare Zero Trust Access**, secured with login policies, bypass rules for analytics scripts, and tested end-to-end.  
The goal: host analytics under a private subdomain, fully secured, but still track visits globally with CSP-compliant script injection.

---

## Project Structure

```plaintext

umami-analytics/
├── cloudflared/
│ ├── config.yml
│ ├── xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json # Tunnel credentials (redacted)
│
├── docker-compose.yml # Umami + Postgres stack
│
├── docs/
│ ├── setup.md # Step-by-step installation
│ ├── troubleshooting.md # All errors + fixes
│ └── policies.md # Cloudflare Access policy design
│
└── README.md # This file

```
---

## Cloudflared Configuration

## 1. Cloudflared Tunnel Setup

To expose Umami Analytics securely, we used **Cloudflare Tunnel** (cloudflared).  
This establishes a persistent outbound-only connection between the server and Cloudflare’s edge.

### Steps Taken

1. **Install cloudflared**  
   ```bash
   sudo apt update && sudo apt install cloudflared -y
    ```
2. Authenticate with Cloudflare
   ```bash
   cloudflared tunnel login
    ```
This opened a browser window and linked the server to our Cloudflare account.
After login, Cloudflare automatically generated a credentials JSON file.
   
3. Created a named tunnel
  ```bash
    cloudflared tunnel create umami
  ```
This produced a UUID-based tunnel ID and credentials JSON file.

4. Move credentials to a secure location
   ```bash
   sudo mv ~/.cloudflared/<UUID>.json /etc/cloudflared/
    ```
5. Create configuration file

### `/etc/cloudflared/config.yml`
```yaml
tunnel: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
credentials-file: /etc/cloudflared/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx.json

ingress:
  - hostname: analytics.example.x
    service: http://localhost:3001
  - service: http_status:404
```

6. Run the tunnel
```yaml

```




## docker-compose.yml
```yaml
version: "3.8"

services:
  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    ports:
      - "3001:3000"
    environment:
      DATABASE_URL: postgresql://umami:xxxxx@db:5432/umami
      HASH_SALT: xxxxxxxxxxxxxxxxxxxxxxxxxxxx
    depends_on:
      - db
    restart: always

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: xxxxx
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: always

volumes:
  db-data:

```

## Cloudflare Zero Trust Setup

Application Name: Analytics

Subdomain: `analytics.example.x`

Session Duration: 24 hours

## Hostnames

`/` → Protected by login policy

`/script.js` → Bypassed (for global tracking)

`/api/*` → Bypassed (for event collection)


## Policies
Order	Name	Action	Rule
1	Analytics Script/API	Bypass	Include: Everyone
2	Only myself can login	Allow	Emails: example@example.x
3	Deny all others	Block	Include: Everyone

# CSP Fix
Umami script was blocked by the server’s Content Security Policy.
Error:
```pgsql
Refused to load the script 'https://analytics.example.x/script.js' 
because it violates Content Security Policy directive "script-src 'self' ..."
```

## Fix (server-side)

Add the analytics domain to CSP:

```ts
// Example in index.ts
scriptSrc: [
  "'self'",
  "'unsafe-inline'",
  "'unsafe-eval'",
  "https://analytics.example.x"
]
```

## Troubleshooting Log
## Troubleshooting

| Problem | Error Message | Cause | Solution |
|---------|---------------|-------|----------|
| Tunnel JSON not found | `couldn't read tunnel credentials ... no such file or directory` | JSON credential file wasn’t generated or misplaced | Ran `cloudflared tunnel login`, copied generated JSON into `/etc/cloudflared/`, fixed file permissions |
| Permission denied on JSON | `couldn't read tunnel credentials ... permission denied` | JSON file owned by root | Changed ownership with `sudo chown root:root` and permissions with `chmod 644` |
| Tunnel connects but site unreachable | No error, but browser can’t reach site | Tunnel ran only in foreground | Installed cloudflared as a service: `sudo cloudflared service install` |
| 530 Error (Cloudflare) | `HTTP/2 530` | DNS record still pointed to old tunnel UUID | Updated DNS CNAME in Cloudflare dashboard to match new UUID |
| Login page bypassed | Dashboard loaded without Cloudflare prompt | Bypass policy applied to `/` instead of `/script.js` + `/api/*` | Corrected public hostname paths and tied Bypass only to script + API |
| Umami not tracking | Browser console: `Refused to load the script ... violates CSP` | Server’s Content Security Policy blocked analytics subdomain | Added `https://analytics.example.x` to CSP `script-src` directive |


Fix: installed Cloudflared as a service:
```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
```

## Problem 4: Cloudflare Access login not enforced

Cause: Bypass rules too broad (applied to `/`).

Fix: split policies → only bypass `/script.js` and `/api/*`, login enforced on `/`.

Problem 5: Umami script not tracking

Cause: CSP blocked external script.

Fix: updated CSP with `https://analytics.example.x`.

---

## Future Plan

Move stack from laptop to Raspberry Pi (24/7 availability).

Configure Pi as lightweight NAS for Postgres storage.

Add nightly backups via `pg_dump`.

Optional: migrate DB to managed cloud provider for resilience.

---

## Security Considerations

No personal data is tracked (GDPR compliant).

Cloudflare Access enforces identity-based login for dashboard.

Bypass paths strictly limited to analytics script + API only.

Tunnel credentials and salts are never committed to GitHub.







