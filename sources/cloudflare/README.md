# Cloudflare

**Version:** 0.1.0
**Backend:** HTTP
**Tables:** 7
**Base URL:** `https://api.cloudflare.com/client/v4`

Query zones, DNS records, accounts, members, Workers scripts, R2 buckets, and
audit logs from Cloudflare.

## Authentication

Requires a `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID`.

```bash
coral source add --interactive cloudflare
```

Or pass credentials directly:

```bash
CLOUDFLARE_API_TOKEN=<token> CLOUDFLARE_ACCOUNT_ID=<account_id> \
  coral source add cloudflare
```

### Creating an API token

1. Go to [dash.cloudflare.com/profile/api-tokens](https://dash.cloudflare.com/profile/api-tokens)
2. Select **Create Token**
3. Use the **Read all resources** template, or create a custom token with:

| Permission | Level |
|---|---|
| Zone — Zone | Read |
| Zone — DNS | Read |
| Account — Account Settings | Read |
| Account — Workers Scripts | Read |
| Account — Workers R2 Storage | Read |
| Account — Audit Logs | Read |

New tokens use the `cfut_` prefix format. Classic `Bearer` tokens also work.

### Finding your Account ID

Your account ID appears on the right-hand sidebar of any zone overview page in
the Cloudflare dashboard. You can also retrieve it via the API:

```bash
curl -s https://api.cloudflare.com/client/v4/accounts \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" | jq '.result[0].id'
```

## Tables

| Table | Description | Required filter |
|---|---|---|
| `zones` | Domains on the account | — |
| `dns_records` | DNS records for a zone | `zone_id` |
| `accounts` | Accounts accessible to the token | — |
| `members` | Account members and their roles | — |
| `workers_scripts` | Workers scripts deployed to the account | — |
| `r2_buckets` | R2 object storage buckets (requires R2 to be enabled) | — |
| `audit_logs` | Account audit log entries | `since`, `before` |

## Quick start

```bash
# Discover tables
coral sql "SELECT table_name, description FROM coral.tables WHERE schema_name = 'cloudflare'"

# List all zones and their status
coral sql "SELECT id, name, status, plan__name FROM cloudflare.zones"

# DNS records for a zone
coral sql "
  SELECT type, name, content, proxied, ttl
  FROM cloudflare.dns_records
  WHERE zone_id = '<your-zone-id>'
  ORDER BY type, name
"

# Find all proxied A records across a zone
coral sql "
  SELECT name, content
  FROM cloudflare.dns_records
  WHERE zone_id = '<your-zone-id>'
    AND type = 'A'
    AND proxied = true
"

# List Workers scripts with last deploy time
coral sql "
  SELECT id, modified_on, handlers, usage_model, last_deployed_from
  FROM cloudflare.workers_scripts
  ORDER BY modified_on DESC
"

# R2 buckets by region
coral sql "
  SELECT name, location, storage_class, creation_date
  FROM cloudflare.r2_buckets
  ORDER BY location, name
"

# Recent audit log entries — since and before are required
coral sql "
  SELECT action__time, actor__email, actor__type, action__type, action__result, resource__type
  FROM cloudflare.audit_logs
  WHERE since = '2026-05-01T00:00:00Z'
    AND before = '2026-05-11T00:00:00Z'
  ORDER BY action__time DESC
  LIMIT 25
"

# Account members
coral sql "
  SELECT user__email, user__first_name, user__last_name, status
  FROM cloudflare.members
  WHERE status = 'accepted'
"
```

## Discovery order

```text
accounts
  → id → CLOUDFLARE_ACCOUNT_ID
    → members
    → workers_scripts
    → r2_buckets
    → audit_logs

zones (scoped to CLOUDFLARE_ACCOUNT_ID)
  → id (zone_id)
    → dns_records
```
