# AGENTS.md — Parspack CDN Public API, for AI agents

This file is the entry point for an AI agent (or any automated client) that
needs to **call** the Parspack CDN public API at `https://my.parspack.com/cdnapi/`.
If you were pointed here to understand the API and integrate against it, start
here. Throughout this file, **the operator** is the human who runs you and owns
the Parspack account and the API token. Several things below can only be done
by the operator in the dashboard; when one blocks you, stop and ask.

The machine-readable contract is the API's OpenAPI 3.0.3 spec, published with
its Redocly docs. **Load it before generating any request** — it is the source
for exact field names, types, enum values and example responses, and this file
does not repeat them. This file covers what the spec cannot show: how the calls
behave, the order to make them in, and how failures look.

## What this API is

A CDN and DNS provider API. A **zone** is one domain on a plan. You order a
zone, point the domain at the name servers the order returns, then manage the
zone's **DNS records** (and DNSSEC), **cache**, **firewall** (access rules,
rate limits, IP reputation, ModSecurity), **SSL**, **load balancers**,
**optimization**, **logs**, and its request rules: **page rules**, **origin
rules** and **transform rules**. **Bulk lists** are account-wide lists of IPs,
user agents, countries or referrers that firewall rules can reference.

```
Base URL : https://my.parspack.com/cdnapi/
Auth     : Authorization: Bearer <token>
Content  : application/json requests and responses
Accept   : application/json
```

Bearer is the only scheme the server accepts. The operator creates the token in
the dashboard with **Generate API token**; it has the form `<id>|<secret>`, and
you send it exactly as issued. Each token carries its own list of permissions
(one per route) and, optionally, a list of IP addresses it may be used from.
The operator chooses both in the dashboard; neither can be changed through this
API, and you cannot create a token yourself.

Before doing real work, confirm the token works by listing zones. A `200` with
`"success": true` means you are ready:

```bash
curl -H "Authorization: Bearer <token>" \
     -H "Accept: application/json" \
     https://my.parspack.com/cdnapi/external/api/v1/zones
```

`Accept-Language: en` returns English `message` strings. Messages are for
humans; do not parse them.

## Before you call anything

1. **A token only works for the routes its permissions cover, and only from
   its allowed IPs.** A missing route permission returns `401` with
   `message: "Token has not access to this action"`. A token used from an IP
   outside its allow-list is treated as **no token at all**: `401` with
   `message: "Unauthorized!"`, the same as a missing, wrong or revoked token.
   If a token that worked elsewhere fails with `Unauthorized!` from CI, a
   container or a serverless function, suspect the egress IP before the token.
   When the request carries `X-Forwarded-For`, that header's whole value is the
   IP that gets checked, so a proxy that appends to it can break an allow-list
   match. Ask the operator to fix the permissions or IPs in the dashboard; you
   cannot do it through this API.
2. **The base URL must keep its trailing slash, and every path you request
   must be relative with no leading slash.** `GET external/api/v1/zones`, not
   `GET /external/api/v1/zones`. A leading slash drops `/cdnapi` from the URL.
   A path the server does not know (wrong prefix, wrong version, wrong method)
   does **not** return `404`; it returns `400` with
   `message: "Operation failed!"`, the same body as a generic failure.
3. **Send `Accept: application/json` on every request.** Without it, a request
   that fails authentication is answered with `400` `"Operation failed!"`
   instead of `401` `"Unauthorized!"`, so a bad token looks like a bad request.
   A validation failure is answered with a `302` redirect to an HTML page
   instead of a `422` JSON body.
4. **Only `active` zones accept calls.** Every path containing `{zone_uuid}`
   checks the zone first. Zone `status` values and what they allow:
   - `active`, `setup_required` — usable. `setup_required` means the zone is
     active but has no main DNS record yet.
   - `pending`, `payment_pending` — the order is not paid yet. Every
     `{zone_uuid}` call, including `GET` and `DELETE` of the zone itself,
     returns `422` with `message: "Service Not Active"` and
     `errors.status: "pending"`.
   - `suspended` — same `422`, with `errors.status: "suspended"`.

   `GET external/api/v1/zones` is the only call that lists zones in every
   status.
5. **`{zone_uuid}` takes the zone's `uuid` or its short `id`.** The zone list
   returns both (`uuid` like `123e4567-…`, `id` like `Q5rGXaKV`); the order
   response returns the `uuid` as `data.zone_id`. Every other id in paths and
   bodies (`bulklist_id`, `load_balance_id`, `modsec_data_ids`, rate limit,
   access rule and load-balancer ids, …) is a short string of the same kind.
   Treat all ids as opaque strings; a numeric id, or one from another account,
   returns `404`.
6. **Some failures come from the account, not from your request.** Retrying
   never fixes them; stop and tell the operator.
   - The account owner has not verified their phone number: `401` with
     `errors: {"mobile_verification": false}` on zone routes.
   - The zone's plan does not include the feature: `403` with
     `message: "service hasn't access to this action"`.
   - The plan's quota for this feature is used up (for example access rules,
     page rules or cache purges): `403` with
     `message: "service access limit has been reached!"`.

## Minimal working flow (order a zone, then add a DNS record)

1. `GET external/api/v1/orders/packages` → pick a plan and billing cycle. The
   `free` plan goes with billing cycle `free`.
2. `POST external/api/v2/orders` with `{ domain, plan, billing_cycle, records }`.
   `records` is the zone's initial DNS, each item
   `{ host, type, ttl, proxy, records: [{ content, … }] }`; `host` is `@` for
   the apex. If you send no MX record the server creates `@` MX pointing to
   `mail.<domain>`, and if you send no NS records it creates the default set.
   Success is `201`.

   **Order once, and read the response before doing anything else.** The zone
   exists as soon as you get a `201`, even when something went wrong after
   that. Check `data.warnings`:
   - empty — the zone was created and the records were stored.
   - "…account balance is not sufficient to activate the service." — the zone
     exists but is unpaid (`status` `payment_pending`), and every
     `{zone_uuid}` call returns `422` until the operator tops up the account
     balance in the dashboard.
   - "…failed to create dns records!" — the zone exists but has no records.
     Add them with the DNS endpoints (step 5).

   Do not repeat the order to "fix" a warning. The domain is now taken, so a
   second order fails with `422` on `domain`. Orders are also rate-limited per
   account and IP to a small number of attempts in a 15-minute window, failed
   attempts included, and an exceeded limit comes back as `400`
   `"Operation failed!"`, not `429`. The same `domain` sent twice within about
   5 seconds returns `423`.

3. Read `data.zone_id` (a UUID) and `data.ns_records` (`ns1` … `ns4`). The
   domain's registrar must be pointed at those name servers; that step happens
   outside this API, so give the name servers to the operator.
4. `GET external/api/v1/zones` → find the zone by `target_domain` and confirm
   `status` is `active` or `setup_required` before making `{zone_uuid}` calls.
5. `POST external/api/v2/zones/{zone_uuid}/dns-records` with
   `{ host, type, ttl, proxy, record: { content } }`. Success is `201`. Note
   that this body takes a single `record` object, while the order and
   `PUT dns-records` take a `records` array.

## Endpoint inventory

Every operation the spec defines. Paths are relative to the base URL — no
leading slash, per rule 2 above. DNS records and orders are `v2`; everything
else is `v1`. These tables are a map so you know what exists; request and
response shapes live in the spec.

### Zones and orders

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones` | List your zones in every status, with `id`, `uuid`, `target_domain`, `status`, `plan` |
| GET | `external/api/v1/zones/{zone_uuid}` | One zone's plan, billing cycle, name-server status and remaining days |
| DELETE | `external/api/v1/zones/{zone_uuid}` | Delete the zone and its DNS records and end its service — irreversible, see gotchas |
| GET | `external/api/v1/orders/packages` | Plans and prices you can order |
| POST | `external/api/v2/orders` | Order a zone — see the flow above |
| GET | `external/api/v1/orders/{zone_uuid}/ns-records` | The name servers to point an existing zone at |
| GET | `external/api/v1/zones/{zone_uuid}/analytics/top-visitors` | Top visitors between `start` and `end` (both required) |
| GET | `external/api/v1/zones/{zone_uuid}/analytics/monthly-traffic-usage` | Traffic used this month |

### DNS records

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v2/zones/{zone_uuid}/dns-records` | List records |
| POST | `external/api/v2/zones/{zone_uuid}/dns-records` | Add one value; appends when that `host` and `type` already exist |
| PUT | `external/api/v2/zones/{zone_uuid}/dns-records` | Change the existing values of one `host` and `type` — see gotchas |
| DELETE | `external/api/v2/zones/{zone_uuid}/dns-records` | Delete one value, or all values of a `host` and `type` — carries a body |
| GET | `external/api/v1/zones/{zone_uuid}/dns-sec` | DNSSEC status |
| PUT | `external/api/v1/zones/{zone_uuid}/dns-sec` | Turn DNSSEC on or off |

A DNS write can fail upstream after validation passes; that is `424`. Re-read
the records with `GET` before retrying, because the change may be partly
applied.

### Zone settings

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/settings/get-antivirus-status` | Antivirus on or off |
| PUT | `external/api/v1/zones/{zone_uuid}/settings/update-antivirus-status` | Turn antivirus on or off |
| PUT | `external/api/v1/zones/{zone_uuid}/settings/developer-mode` | Turn developer mode on or off |
| PUT | `external/api/v1/zones/{zone_uuid}/settings/maintenance-mode` | Turn maintenance mode on or off |
| PUT | `external/api/v1/zones/{zone_uuid}/settings/origin-offline` | Turn origin offline mode on or off |
| PUT | `external/api/v1/zones/{zone_uuid}/settings/query-string` | Turn query string ignoring on or off |
| GET | `external/api/v1/zones/{zone_uuid}/https-convertor` | HTTPS convertor on or off |
| PUT | `external/api/v1/zones/{zone_uuid}/https-convertor` | Change it |
| GET | `external/api/v1/zones/{zone_uuid}/edge-to-upstream-connection` | Protocol from the CDN to your origin (`auto`, `http`, `https`) |
| PUT | `external/api/v1/zones/{zone_uuid}/edge-to-upstream-connection` | Change it |
| GET | `external/api/v1/zones/{zone_uuid}/www-redirection` | `www` redirect setting |
| PUT | `external/api/v1/zones/{zone_uuid}/www-redirection` | Change it |
| GET | `external/api/v1/zones/{zone_uuid}/web-socket` | WebSocket support |
| PUT | `external/api/v1/zones/{zone_uuid}/web-socket` | Change it |
| GET | `external/api/v1/zones/{zone_uuid}/upstream-errors` | Upstream errors setting on or off |
| PUT | `external/api/v1/zones/{zone_uuid}/upstream-errors` | Change it |
| GET | `external/api/v1/zones/{zone_uuid}/ddos-actions` | DDoS action and its trust and ban times |
| PUT | `external/api/v1/zones/{zone_uuid}/ddos-actions` | Change them |

### Cache

| Method | Path | Purpose |
| --- | --- | --- |
| DELETE | `external/api/v1/zones/{zone_uuid}/cache` | Purge the whole zone's cache — asynchronous, see gotchas |
| GET | `external/api/v1/zones/{zone_uuid}/cache` | List purge jobs |
| GET | `external/api/v1/zones/{zone_uuid}/cache/{id}` | One purge job |
| GET | `external/api/v1/zones/{zone_uuid}/cache/settings` | Current cache settings |
| PUT | `external/api/v1/zones/{zone_uuid}/cache/ttl` | Set the edge cache TTL |
| PUT | `external/api/v1/zones/{zone_uuid}/cache/rule` | Set the cache rule (`cdn-no-caching`, `cdn-static-caching`, `cdn-smart-caching`, `cdn-always-caching`) |
| PUT | `external/api/v1/zones/{zone_uuid}/cache/user-agent` | Turn caching per user agent on or off |

### Firewall

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/countries` | Countries and their ids, for country rules |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/access-management` | List access rules; `page`, `per_page` (max 100) |
| POST | `external/api/v1/zones/{zone_uuid}/firewalls/access-management` | Add an access rule |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/access-management/{access_management_id}` | One access rule |
| PUT | `external/api/v1/zones/{zone_uuid}/firewalls/access-management/{access_management_id}` | Replace an access rule |
| DELETE | `external/api/v1/zones/{zone_uuid}/firewalls/access-management/{access_management_id}` | Delete an access rule |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/ip-reputation` | IP reputation settings |
| PUT | `external/api/v1/zones/{zone_uuid}/firewalls/ip-reputation` | Change them |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/rate-limit` | List rate-limit rules |
| POST | `external/api/v1/zones/{zone_uuid}/firewalls/rate-limit` | Add a rate-limit rule |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/rate-limit/{rate_limit_rule_id}` | One rate-limit rule |
| PUT | `external/api/v1/zones/{zone_uuid}/firewalls/rate-limit/{rate_limit_rule_id}` | Replace a rate-limit rule |
| PUT | `external/api/v1/zones/{zone_uuid}/firewalls/rate-limit/{rate_limit_rule_id}/update-priority` | Move a rule to another priority |
| DELETE | `external/api/v1/zones/{zone_uuid}/firewalls/rate-limit/{rate_limit_rule_id}` | Delete a rate-limit rule |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security` | Managed ModSecurity rules and which are on |
| POST | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security` | Set the managed rules that are on — full replacement, see gotchas |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/data` | List custom ModSecurity data |
| POST | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/data` | Add custom data |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/data/{id}` | One custom data item |
| PUT | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/data/{id}` | Change custom data |
| DELETE | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/data/{id}` | Delete custom data |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/rules` | List custom ModSecurity rules |
| POST | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/rules` | Add a custom rule |
| GET | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/rules/{mod_sec_rule}` | One custom rule |
| PUT | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/rules/{mod_sec_rule}` | Change a custom rule |
| DELETE | `external/api/v1/zones/{zone_uuid}/firewalls/mod-security/rules/{mod_sec_rule}` | Delete a custom rule |

### Page rules

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/page-rules` | List page rules |
| POST | `external/api/v1/zones/{zone_uuid}/page-rules` | Create a page rule |
| GET | `external/api/v1/zones/{zone_uuid}/page-rules/{page_rule}` | One page rule |
| PUT | `external/api/v1/zones/{zone_uuid}/page-rules/{page_rule}` | Rewrite a page rule — see gotchas |
| DELETE | `external/api/v1/zones/{zone_uuid}/page-rules/{page_rule}` | Delete a page rule |

### Origin rules

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/origin-rules` | List origin rules |
| POST | `external/api/v1/zones/{zone_uuid}/origin-rules` | Create an origin rule |
| GET | `external/api/v1/zones/{zone_uuid}/origin-rules/{origin_rule}` | One origin rule |
| PUT | `external/api/v1/zones/{zone_uuid}/origin-rules/{origin_rule}` | Rewrite an origin rule |
| PUT | `external/api/v1/zones/{zone_uuid}/origin-rules/{origin_rule}/toggle-status` | Enable or disable an origin rule (`enabled`) |
| DELETE | `external/api/v1/zones/{zone_uuid}/origin-rules/{origin_rule}` | Delete an origin rule |

### Transform rules

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/transform-rules` | List transform rules |
| POST | `external/api/v1/zones/{zone_uuid}/transform-rules` | Create a transform rule that changes request or response headers |
| GET | `external/api/v1/zones/{zone_uuid}/transform-rules/{transform_rule}` | One transform rule |
| PUT | `external/api/v1/zones/{zone_uuid}/transform-rules/{transform_rule}` | Rewrite a transform rule |
| PUT | `external/api/v1/zones/{zone_uuid}/transform-rules/{transform_rule}/toggle-status` | Enable or disable a transform rule (`enabled`) |
| DELETE | `external/api/v1/zones/{zone_uuid}/transform-rules/{transform_rule}` | Delete a transform rule |

Origin and transform rules choose which requests they apply to with
`rule_items`: a list of condition groups. Groups are ORed together, and the
conditions inside one group are ANDed.

### SSL

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/ssl/certificates` | List certificates; `page`, `per_page` (max 100), `domain_filter` |
| GET | `external/api/v1/zones/{zone_uuid}/ssl/min-tls-version` | Minimum TLS version |
| PUT | `external/api/v1/zones/{zone_uuid}/ssl/min-tls-version` | Change it |
| GET | `external/api/v1/zones/{zone_uuid}/hsts` | HSTS settings |
| PUT | `external/api/v1/zones/{zone_uuid}/hsts` | Change them |

### Load balancing

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/load-balance` | List load balancers |
| POST | `external/api/v1/zones/{zone_uuid}/load-balance` | Create a load balancer with its servers |
| GET | `external/api/v1/zones/{zone_uuid}/load-balance/{load_balance}` | One load balancer |
| PUT | `external/api/v1/zones/{zone_uuid}/load-balance/{load_balance}` | Change a load balancer's settings (not its servers) |
| DELETE | `external/api/v1/zones/{zone_uuid}/load-balance/{load_balance}` | Delete a load balancer |
| GET | `external/api/v1/zones/{zone_uuid}/load-balance-server` | List one load balancer's servers; `load_balance_id` query is required |
| POST | `external/api/v1/zones/{zone_uuid}/load-balance-server` | Add a server to a load balancer |
| GET | `external/api/v1/zones/{zone_uuid}/load-balance-server/{load_balance_server}` | One server |
| PUT | `external/api/v1/zones/{zone_uuid}/load-balance-server/{load_balance_server}` | Change a server |
| DELETE | `external/api/v1/zones/{zone_uuid}/load-balance-server/{load_balance_server}` | Remove a server |

A load balancer is used by setting its id as `load_balance_id` on a DNS record.

### Optimization

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/optimization/status` | Image minification, WebP conversion and HTML/CSS/JS minification |
| PUT | `external/api/v1/zones/{zone_uuid}/optimization/update` | Change them; all fields are required |

### Logs

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/zones/{zone_uuid}/report/access-log` | Access log |
| GET | `external/api/v1/zones/{zone_uuid}/report/security-log` | Security log |
| GET | `external/api/v1/zones/{zone_uuid}/report/error-log` | Error log |
| GET | `external/api/v1/zones/{zone_uuid}/report/waf-log` | WAF log |

### Bulk lists

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `external/api/v1/user/bulklists` | List your bulk lists; `page`, `per_page` (max 100) |
| POST | `external/api/v1/user/bulklists` | Create a bulk list |
| GET | `external/api/v1/user/bulklists/{bulklist_id}` | One bulk list |
| PUT | `external/api/v1/user/bulklists/{bulklist_id}` | Replace a bulk list, including all its items |
| DELETE | `external/api/v1/user/bulklists/{bulklist_id}` | Delete a bulk list |

## Gotchas that will break a naive client

These are the traps a generic client built purely from guessing conventions
will fall into. Full detail is in the spec; this is the "read this or you
will lose an hour" version.

| Area | Gotcha |
| --- | --- |
| Changes that affect live traffic | Only change a zone the operator explicitly named. Maintenance mode, origin offline mode, developer mode, DNS and DNSSEC changes, access rules, rate limits, ModSecurity, page, origin and transform rules and cache purges all change what visitors get, immediately. Never try them on a zone just to see what happens, and never on one you found incidentally in the list. |
| Repeated writes | Most writes take a short lock per account, zone and target. The same write sent again within about 2 seconds (5 for orders) returns `423` `"You have another running process!"`, even if the first attempt failed validation. Wait and retry; this is one of the few errors where retrying the identical request helps. DNS record writes, bulk lists, `PUT .../cache/ttl` and zone deletion are not locked. |
| `POST external/api/v2/orders` | Creates a billable service, is not idempotent and is tightly rate-limited. Only order the domain, plan and billing cycle the operator named. A `201` with `warnings` still created the zone; repeating the order fails because the domain is taken. See the flow above. |
| `DELETE external/api/v1/zones/{zone_uuid}` | Irreversible: the service is ended, unpaid invoices for it are removed, and its DNS records are deleted. Only call it for a zone the operator explicitly named for deletion, never for one discovered incidentally (e.g. "first in the list"). It also needs an `active` zone, so an unpaid zone cannot be deleted through the API. |
| `DELETE …/dns-records` | Carries a JSON body (`host`, `type`, optional `record`). Many HTTP clients drop bodies on `DELETE` by default, which surfaces as a `422`. Leaving out `record` deletes **every** value of that `host` and `type`. |
| `POST` vs `PUT …/dns-records` | `POST` appends a value when the `host` and `type` already exist; it never replaces. `PUT` edits the existing values in place and cannot add or remove any: send the same number of `records` that `GET` returned. To add or remove a value, `POST` or `DELETE` it. |
| Rate-limit `value` | A URL pattern **without** a scheme that contains the zone's own domain followed by `/`: `example.com/*` or `shop.example.com/api/*`. `https://example.com/*` is rejected, as is a pattern for another domain or one starting with `**`. |
| Rate-limit intervals | When `static_interval_type` or `dynamic_interval_type` is `second`, the matching interval must be at least `10`. |
| Access rule and bulk list values | The meaning of `value` (and of bulk list `items`) depends on `type`: `ip` is an IP or CIDR, `country` is a country **id** from `firewalls/countries` (not an ISO code), `user_agent` and `referral` are strings up to 191 characters, and a `referral` must not start with a scheme like `https://`. |
| Page rules | `value` depends on `type`. For `url` (the default) it is a URL pattern **without** a scheme that contains the zone's own domain, such as `example.com/blog/*`; `https://…`, a pattern starting with `**`, or one that matches none of the zone's hostnames is rejected. For `user_agent` the `operator` is required and is one of `equals`, `not_equals`, `contains`, `not_contains`; for `url` it must be `pattern` or left out. Names are unique per zone. Creating a page rule past the plan's limit returns `403` `"service access limit has been reached!"`. |
| Page rule `PUT` | Rewrites the whole rule from the body. Anything you leave out goes back to its default (`firewall_status` true, header, cookie and IP lists empty, redirection and port destination removed), so send every field you want to keep. |
| Origin rules | Send the field that matches `type`: `upstream_ip` for `upstream` (the default), `port` for `port`, `load_balance_id` for `load_balance`, which must be a load balancer of the same zone. The other two are not stored, but if you send them they must still be valid. Names are unique per zone. |
| Transform rules | `transform_rule_values` holds `request_headers` and `response_headers` lists (up to 20 each). Each entry is `{ header_name, action }` with `action` `modify` or `delete`; `header_value` is required for `modify`. |
| Origin and transform rule toggles | `toggle-status` changes only `enabled`. A normal `PUT` needs the full rule, including `rule_items`. |
| Access rule `PUT` | Full replacement: send `type`, `action`, `status` and the `value` or `bulklist_id` again, even if unchanged. |
| ModSecurity managed rules | `POST firewalls/mod-security` sets the **complete** list of managed rules that are on. Rules you leave out are turned off, and `"modsec_rules": []` turns all of them off. Read the current list with `GET` first and send it back with your change. |
| ModSecurity custom rules | `PUT` replaces both `rule_value` and `modsec_data_ids`. Leaving out `rule_value` (or sending `null`) **wipes the rule text**, so send the current value back even when you only rename the rule. `modsec_data_ids` must always be present; `[]` unlinks everything. Rule text and data values are **base64**, not raw ModSecurity source. |
| Load balancer servers | `POST load-balance` creates the balancer and its servers together. `PUT load-balance/{load_balance}` changes only the balancer's settings; add, change or remove servers with the `load-balance-server` routes. |
| Cache purge | `DELETE …/cache` returns as soon as the purge job is queued, not when the cache is empty. Poll `GET …/cache` or `GET …/cache/{id}`. Purges count against the plan's quota; an exhausted quota is `403` `"service access limit has been reached!"`. |
| `min_tls_version` | Send a JSON string (`"1.2"`), not a number. |
| Log pagination | Log routes page with `page` and `step` (`10`, `25`, `50` or `100`), not `per_page`. Time filters (`from`, `to`, and analytics `start`, `end`) accept `Y-m-d`, `Y-m-d H`, `Y-m-d H:i` or `Y-m-d H:i:s`. |
| Booleans | Send JSON `true` / `false`. |

## Errors

Every failure is a JSON envelope:

```json
{ "success": false, "message": "…", "errors": { "field": ["…"] } }
```

There is no machine-readable error code. Branch on the HTTP status plus the
keys inside `errors`, never on `message`: messages are translated and change
with `Accept-Language`. `errors` is an object for validation failures and for
the few errors that carry detail (`status`, `mobile_verification`), and an
empty array `[]` otherwise.

| HTTP | Typical `message` | What it means for you |
| --- | --- | --- |
| 400 | `Operation failed!` | Several different things share this: a path or method the server does not know (rule 2), the order rate limit (see the flow), a request sent without `Accept: application/json` that failed authentication (rule 3), or an operation the server could not complete. Check the path and the headers first. Retrying with backoff only makes sense for a path you know exists |
| 401 | `Unauthorized!` | Missing, wrong or revoked token, **or** a good token from an IP outside its allow-list (rule 1). Retrying never helps |
| 401 | `Token has not access to this action` | The token lacks this route's permission. The operator has to add it in the dashboard |
| 401 | `service hasn't access to this action`, `errors.mobile_verification: false` | The account owner has not verified their phone number. The operator has to do it; retrying never helps |
| 403 | `service hasn't access to this action` | The zone's plan does not include this feature. Retrying never helps; the operator has to change the plan |
| 403 | `service access limit has been reached!` | The plan's quota for this feature is used up (access rules, page rules, cache purges). Retrying never helps until something is deleted or the plan changes |
| 403 | `user hasn't access to this action` | Returned by some routes when the resource id exists but does not belong to this zone |
| 404 | `Service Not Found` or `Not found!` | The zone or resource does not exist, belongs to another account, or the id is malformed (for example numeric). An unknown zone uuid gives `Service Not Found`; a malformed zone id or an unknown resource id gives `Not found!`. Deleted zones are `404` too |
| 422 | Summary of the first validation error | Input the server rejected; field messages are under `errors`. Fix the payload; retrying it unchanged never helps |
| 422 | `Service Not Active`, `errors.status: "pending"` or `"suspended"` | The zone is not active (rule 4). Nothing on the zone works until it is paid or unsuspended |
| 423 | `You have another running process!` | The same write is still locked (see gotchas). Wait about 2 seconds (5 for orders) and retry |
| 424 | Operation-specific | A DNS change failed after validation. Re-read the records before retrying |
| 500 | Varies | A server-side fault. Rare, since most unexpected failures surface as `400`. Retry with backoff |

Three caveats before you write a parser.

- Two different `401`s and two different `403`s share a status code; tell
  them apart by `message` wording only when you have to explain the failure to
  the operator, and by the presence of `errors.mobile_verification` otherwise.
  None of them is fixed by retrying.
- A missing `Accept: application/json` header turns an authentication failure
  into `400` `"Operation failed!"` and a validation failure into a `302`
  redirect with an HTML body instead of a `422` (rule 3). Set the header
  before you interpret any status code.
- Success responses are the same envelope with `success: true` and a `data`
  key. Paginated lists add `meta` (`current_page`, `last_page`, `per_page`,
  `total`) and sometimes `links`; read `meta` and do not require `links`.

Success codes: `POST external/api/v2/orders`, `POST …/dns-records` and every
`POST` that creates a resource (access rules, rate-limit rules, ModSecurity
custom data and rules, load balancers and their servers, page, origin and
transform rules, bulk lists) are `201`.
Everything else is `200`, including every `DELETE`, `PUT …/dns-records` and
`POST firewalls/mod-security`. No endpoint answers with an empty body. Treat
any 2xx as success rather than matching an exact code.

If you discover a new field, error shape, or behavior while integrating, report
it to the operator rather than keeping private undocumented knowledge.
