# AGENTS.md — Abrha Cloud Public API, for AI agents

This file is the entry point for an AI agent (or any automated client) that
needs to **call** the `my.abrha.net` public API — not to run this repo's test
suite. If you were pointed at this repo to understand the API and integrate
against it, start here.

The machine-readable contract is
[`server-api.openapi`](server-api.openapi) (OpenAPI 3.0.3), generated from the
server code. **Load it before generating any request** — it is the only source
for exact field names, types and enum values, and this file does not repeat
them.

It is not, however, correct everywhere. Where this file and the spec disagree,
the spec wins **except** for the specific defects listed under
["Where the spec is wrong"](#where-the-spec-is-wrong) — those are confirmed
against the running server, and following the spec there will produce failing
requests.

## What this API is

A DigitalOcean-style cloud provider API: pick a **region**, a **size**
(hardware plan) and an **image** (OS), create a **VM**. VMs can join a
**VPC**, be protected by a **firewall**, and take **SSH keys**. Anything
long-running returns an **action** you poll to completion.

```
Base URL : https://my.abrha.net/cserver/api/public/v1/
Auth     : Authorization: Bearer <token>
Content  : application/json requests and responses
```

Bearer is the only scheme the server accepts. `Basic` is not supported — the
token is read from the `Authorization: Bearer` header and nothing else, so a
`Basic` credential is treated as no credential at all.

The spec's `servers` entry says `https://my.parspack.com/cserver`, which is the
same API under a different brand. If you were given a token for one of the two
hosts, use that host; the paths below are identical on both.

## Before you call anything

1. **Tokens are locked to the IP addresses registered for them.** A valid,
   unexpired token presented from an unregistered IP is rejected with `401`.
   The two 401 causes are distinguishable by body shape: a wrong IP returns
   `{"id":"unauthorized", ...}`, while a missing, malformed, expired or revoked
   token (and a token lacking the required ability) returns the framework
   envelope `{"success":false,"code":401,"error_code":30002,"message":"..."}`
   with no `id` at all. If you are running from CI, a container, a serverless
   function, or any host with a changing egress IP, an `id: unauthorized` 401
   means the IP — not the token value. Ask the operator to register the IP in
   the dashboard; you cannot do it through this API.
2. **The base URL must keep its trailing slash, and every path you request
   must be relative with no leading slash.** `GET regions`, not `GET /regions`.
   A leading slash resets the URL to the domain root and silently drops
   `/cserver/api/public/v1`, which also surfaces as a confusing 401 or an HTML
   body.
3. Send `Accept: application/json`. A misrouted or unauthenticated request can
   come back as an HTML page instead of a JSON error body.
4. Resource ids are inconsistently typed — sometimes a JSON string, sometimes
   a JSON number, even for the same field on different endpoints. VM,
   firewall, VPC and snapshot ids are strings, while image, size, action and
   SSH-key ids are integers. Coerce to string before comparing; never rely on
   strict equality against a number. Reserved IPs are the odd one out: the
   object carries a numeric `id`, but every path addresses them by IP
   address, so the `id` is not what you put in the URL.

## Minimal working flow (create a VM)

1. `GET regions` → pick a `slug` where `available == true`.
2. `GET sizes?page=1&per_page=200` → pick a `slug` whose own `regions` array
   contains that region. This endpoint is paginated with a default
   `per_page` of 20, which hides most of the catalog, so without `per_page`
   you will conclude that no size fits. Only fall back to the region's own
   `sizes` array for sizes that carry no `regions` of their own. Compare
   region identifiers case-insensitively: the server lowercases region slugs,
   while the spec's examples show them mixed-case (`Tehran2`, `London1`).
3. `GET images` → pick a `slug` where `status == "available"`.
4. `POST vms` with `{ name, region, size, image, ssh_keys }` — `region`,
   `size` and `image` are slugs, not display names. Success is `201`. The
   endpoint description claims `202 Accepted`; that text is wrong, though the
   meaning it conveys is right — the VM is not ready when you get the response.

   **The response never contains the root password, and nothing you can call
   headlessly will hand it to you later.** For security the password is shown
   only in the web dashboard. If you (the agent) need to log in to the VM
   afterwards, this is the only moment to arrange it: upload a public key with
   `POST account/keys` first and pass its `id` or fingerprint in `ssh_keys`
   here — both forms are accepted. A VM created without `ssh_keys` is reachable
   only by a human who reads the password from the panel. **The
   `password_reset` flow is not a fallback you can run on your own.** Its
   first step sends a one-time code by SMS to the account owner's phone, and
   its second step rejects the request without that code (`verify_code` is
   required). An agent has no way to read an SMS, so unless a human is
   standing by to relay the code, a VM created without `ssh_keys` stays out of
   your reach for good. Plan the key in at creation time; do not create the VM
   first and expect to recover access later.

   Entries in `ssh_keys` that match no key of yours are **silently dropped** —
   the lookup is a plain "where id or fingerprint is in this list", with no
   error for the ones it cannot find. A single typo therefore produces a
   perfectly healthy VM that nobody can log into. Re-read
   `GET account/keys` and confirm every value you are about to send exists.

5. Read `links.actions[0].id` from the response (or parse the last path
   segment of `links.actions[0].href` if `id` is absent).
6. `GET actions/{actionId}` repeatedly — **unless the id is `0`**. An action
   with `id: 0` is synthetic: the work already finished synchronously and
   there is nothing to poll (`GET actions/0` will not find it). VM creation
   always returns a real id; see "VM actions" for the types that return `0`.
   `status` has exactly three values:
   - `completed` — done, succeeded
   - `errored` — done, failed
   - `in-progress` — keep polling, with backoff (2s, 5s, 10s, …); a VM create
     can take up to ~3 minutes.

   Do not branch on `success`, `failed` or `error`. The spec shows
   `status: success` in an example, but that value contradicts its own enum and
   the server never sends it.

A region/size pair can be individually valid yet incompatible together, and
there is no endpoint that validates the pair up front. `POST vms` answers
either `404` `not_found` ("The size not found for the selected region") or
`422` `unprocessable_entity` (`LOCATION_NOT_VALID_FOR_PACKAGE`), depending on
where the mismatch is caught — treat both as "invalid pair", not as a missing
resource. Take the pair from the catalog (step 2) rather than guessing: a pair
the catalog does not list will never succeed, no matter how many times you
retry it. Retrying across catalog-valid combinations is still worth it for
quota, capacity and bad-image failures, so do not treat a single non-2xx as
"VM creation is broken".

## Endpoint inventory

Every operation the spec defines. Paths are relative to the base URL — no
leading slash, per rule 2 above. This table is a map so you know what exists;
request and response shapes live in the YAML.

### Catalog — read these before creating anything

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `sizes` | Hardware plans. Paginated, small default page — see gotchas |
| GET | `regions` | Data-center regions |
| GET | `images` | OS images |

### VMs

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `vms` | Create one VM (`name`) or several (`names` array). The prose says "up to ten" but the server enforces no cap — stay at ten anyway |
| GET | `vms` | List your VMs |
| GET | `vms/{id}` | Retrieve one VM |
| DELETE | `vms/{id}` | Destroy a VM — irreversible |
| GET | `vms/{id}/backups` | Backups taken of this VM |
| GET | `vms/{id}/snapshots` | Snapshots created from this VM |
| GET | `vms/{id}/traffic` | Daily traffic usage; optional `date` as `Y-m-d`, defaults to today |
| POST | `vms/{id}/password/reset` | Second step of the password reset flow — see gotchas |

### VM actions

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `vms/{id}/actions` | Initiate an action; `type` is required |
| GET | `vms/{id}/actions` | List actions for this VM |
| GET | `vms/{id}/actions/{actionId}` | Retrieve one action for this VM |

Supported `type` values, from the spec's own enum: `reboot`, `power_on`,
`power_off`, `resize`, `rebuild`, `rename`, `restore`, `snapshot`,
`password_reset`, `enable_ipv6`, `enable_backups`, `disable_backups`,
`change_backup_policy`. `power_cycle`, `shutdown` and `change_kernel` are **not**
in the enum — do not send them just because a sibling provider supports them.

Some types need extra fields in the same body: `name` for `rename` and
`snapshot`; `size` **and** boolean `disk` for `resize` — both are required
(`disk: true` resizes the disk to the new plan's size, `disk: false` keeps the
current one; omitting `disk` is a `422`);
`backup_policy` for `change_backup_policy`; `image` for `rebuild` and
`restore` — but the two disagree on what `image` means. `rebuild` takes an
image slug or id, while `restore` insists on an **integer backup id** sent as
a JSON number (the numeric string `"123"` is rejected) and answers `422` for
anything else.

Not every type is asynchronous. `enable_backups`, `disable_backups`, `rename`,
`password_reset`, `change_backup_policy` and `enable_ipv6` complete inside the
request and return a **synthetic** action: `id: 0`, `status: completed`. Do not
poll `actions/0` or `vms/{id}/actions/0` — it does not exist. Re-read the VM
(or, for `password_reset`, proceed to the second step) instead. `reboot`,
`power_on`, `power_off`, `resize`, `rebuild`, `restore` and `snapshot` return a
real action id you poll as usual.

### Actions (account-wide)

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `actions` | List every action on the account |
| GET | `actions/{actionId}` | Poll a single action to completion |

### Snapshots

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `snapshots` | List all snapshots |
| GET | `snapshots/{snapshotId}` | Retrieve one snapshot |
| DELETE | `snapshots/{snapshotId}` | Delete a snapshot |

### Firewalls

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `firewalls` | List firewalls |
| POST | `firewalls` | Create a firewall |
| GET | `firewalls/{firewall_id}` | Retrieve one firewall |
| PUT | `firewalls/{firewall_id}` | Full replacement — see gotchas |
| DELETE | `firewalls/{firewall_id}` | Delete a firewall |
| POST | `firewalls/{firewall_id}/vms` | Attach VMs |
| DELETE | `firewalls/{firewall_id}/vms` | Detach VMs |
| POST | `firewalls/{firewall_id}/rules` | Add rules incrementally |
| DELETE | `firewalls/{firewall_id}/rules` | Remove rules — carries a body |

### VPCs

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `vpcs` | Create a VPC; `ip_range` is a bare address — see gotchas |
| GET | `vpcs` | List VPCs |
| GET | `vpcs/{vpcId}` | Retrieve one VPC |
| DELETE | `vpcs/{vpcId}` | Delete a VPC. `204` on success, `400` with body `{}` if the server declined — see gotchas |
| GET | `vpcs/{vpcId}/members` | Resources attached to the VPC |

### Reserved IPs

| Method | Path | Purpose |
| --- | --- | --- |
| POST | `reserved_ips` | Reserve an IP |
| GET | `reserved_ips` | List reserved IPs |
| GET | `reserved_ips/{reservedIp}` | Retrieve one; address it by IP, not by the object's numeric `id` |
| DELETE | `reserved_ips/{reservedIp}` | Release a reserved IP |
| POST | `reserved_ips/{reservedIp}/actions` | `assign` (needs `vm_id`) or `unassign`. Synchronous — see below |
| GET | `reserved_ips/{reservedIp}/actions/{actionId}` | Stub — returns a fabricated `completed` action for any integer id |

Reserved-IP actions are **not** real actions. `POST` performs the attach or
detach inside the request and answers `200` with an action object whose `id`
is a random number and whose `status` is always `completed` (the `type` says
`assign_ip` even for `unassign`). The `GET` endpoint does not look anything up:
it echoes whatever `actionId` you pass as a `completed` `assign_ip` action, so
it cannot tell you whether anything happened. There is no list endpoint either.
To confirm the result, read `GET reserved_ips/{reservedIp}` and check its `vm`
field; a failed assign surfaces as a non-2xx on the `POST` itself.

### SSH keys

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `account/keys` | List keys |
| POST | `account/keys` | Upload a public key |
| GET | `account/keys/{idOrFingerPrint}` | Retrieve by id or fingerprint |
| PUT | `account/keys/{idOrFingerPrint}` | Rename |
| DELETE | `account/keys/{idOrFingerPrint}` | Delete |

## Gotchas that will break a naive client

These are the traps a generic client built purely from guessing conventions
will fall into. Full detail is in the YAML; this is the "read this or you
will lose an hour" version.

| Area | Gotcha |
| --- | --- |
| `GET sizes` pagination | Paginated with a small default page, so most of the catalog is missing unless you send `per_page` (e.g. `sizes?page=1&per_page=200`). A size absent from page 1 looks like it doesn't exist. |
| `size` × `region` | Each size lists the `regions` it can be ordered in; any other pairing fails with either `404` `not_found` ("The size not found for the selected region") or `422` `unprocessable_entity` (`LOCATION_NOT_VALID_FOR_PACKAGE`, e.g. `{"id":"unprocessable_entity","code":50039}`). Both mean the same thing. Retrying an unlisted pair never helps. |
| VPC `ip_range` | Asymmetric: **write a bare address, read CIDR**. `POST vpcs` requires a plain IP and rejects `10.10.42.0/24`. The accepted form is much narrower than "a private range": the value must match `10.10.N.0`, `172.16.N.0` or `192.168.N.0` exactly, with `N` in 0–255 and **the last octet always `0`**. So `10.10.42.0` works, while `10.0.42.0` (wrong second octet) and `10.10.42.5` (non-zero last octet) are both `422`. Reads return the same field in CIDR form (`10.10.42.0/24`) — strip anything after `/` before comparing. |
| Firewall rules `ports` | Asymmetric, and the opposite of what the spec's response schema implies: **send a single numeric value** (`22`; the numeric string `"22"` also passes), **receive a string** (`"22"`). One port only — `"8000-9000"` and arrays are rejected on write even though the response description advertises ranges. To open several ports, send one rule per port. Omitting `ports` (or sending `0`/`null`) means **all ports** for that protocol — never omit it by accident. |
| Firewall rules `sources.addresses` | Here CIDR **is** accepted (`0.0.0.0/0` = anywhere) — the opposite convention from VPC `ip_range`. IPv4 only, and any CIDR must be network-aligned: `10.0.0.0/8` is valid, `10.0.0.5/8` is not. Every rule must carry a `sources` (inbound) or `destinations` (outbound) object, but `addresses` inside it is optional, and leaving it out creates the rule **with no address restriction** — open to the whole internet, the same trap as omitting `ports`. |
| `PUT firewalls/{id}` | Full replacement, not a patch, and `name` is **required** even if unchanged. Every existing rule is deleted and only the rules in the body are recreated, so omitting `inbound_rules` clears them and can lock out every attached VM. Omitting `vm_ids` **detaches every VM** from the firewall (the attachment list is synced to an empty set). Always send `name`, `vm_ids`, `inbound_rules` and `outbound_rules` in full, or use `POST .../rules`, `DELETE .../rules`, `POST .../vms`, `DELETE .../vms` for incremental changes. |
| `DELETE firewalls/{id}/rules` | Carries a request body identifying which rules to remove — unusual for `DELETE`; many HTTP clients drop bodies on `DELETE` by default. A near-miss on the rule object silently removes nothing. |
| Rename an SSH key | `PUT account/keys/{id}` accepts `name` either as a JSON body field or as a query parameter (`?name=...`); the spec documents the query form. The name must be unique across your own keys, so a rename that collides fails validation. |
| SSH key material | `public_key` is parsed by a general-purpose key loader, so PEM/PKCS8 public keys pass validation too — but send single-line OpenSSH `authorized_keys` format (`ssh-ed25519 AAAA... comment`), because that is what gets embedded in the VM. A **private** key is rejected (it does not load as a public key). A bad key fails in one of two ways: unparseable at validation time is a `422`, while a key that validates but cannot be fingerprinted comes back as `403` `server_error` with `message: "The provided public key is invalid."` — treat both as "fix the key", not as a server fault. |
| Upload the same key twice | `POST account/keys` is deduplicated by key material. Re-uploading a public key you already have returns the **existing** key — same `id`, its original `name` — with a `201` and no error. It is not a rename; use `PUT account/keys/{id}` for that. |
| SSH key `name` | Validated twice, and the second layer is stricter than the first: it allows only letters, digits, space, `-` and `_` (**no dots**) and caps the length at 30. A name that clears the documented rules can still come back as a `422` from the inner check. |
| Reserved IPs path | The path is `reserved_ips` with an underscore, not `reserved-ips`, and every path segment is the IP address itself (`reserved_ips/45.10.63.85`). The object's numeric `id` is **not** addressable — passing it gives you a 404 or a validation error. |
| Firewalls | Deny-by-default. Omitting an SSH (port 22) rule from `inbound_rules` locks out every VM the firewall attaches to. |
| `DELETE vms/{id}` | Irreversible — no soft delete, no undo. Only call it for a VM the caller explicitly named for destruction; never on a VM discovered incidentally (e.g. "first in the list"). |
| VM actions (`POST vms/{id}/actions`) | Disruptive (drops connections). Only call against a VM the caller explicitly named. Use only the `type` values listed under "VM actions" above; anything outside that enum is rejected. |
| No password in `POST vms` | The create response (and every later `GET vms/{id}`) omits the root password on purpose; it is visible only in the web dashboard. There is no "get credentials" endpoint. The only headless way in is to pass `ssh_keys` at creation (upload the public key with `POST account/keys` first), and unknown entries there are dropped without an error. Do not tell the caller "the password is in the response" — it is not — and do not create a VM for an agent to manage without `ssh_keys`. |
| Numeric `image` on `POST vms` | A numeric value is **not** an image id: the server branches on `is_numeric($image)` and resolves the number as a **backup id**, so it builds the VM from one of your backups and answers `404` when no such backup exists. To create from a catalog image always send the `slug` (`ubuntu24-cloudinit-qcow2`), never the `id` you read from `GET images` — despite the spec describing `image` as "the image ID … or the unique slug". Numeric is only correct when you genuinely mean "restore this backup". |
| Locked VMs | Check `vm.locked` before mutating — a locked VM has an operation in flight and rejects further changes with `422` `unprocessable_entity`. Wait for the in-flight action to leave `in-progress` and retry; this is one of the few 422s where retrying does help. |
| Password reset | Two steps, and the second one is **not headless**. `POST vms/{id}/actions` with `type: password_reset` sends an SMS code to the account owner's phone and returns a synthetic action (`id: 0`, `status: completed`, `description` contains the URL of step two) — do not poll it. `POST vms/{id}/password/reset` then requires that `verify_code` (max 5 chars) and returns the new password. The feature is account-gated: `403` `feature_is_not_active` means the operator has to enable it. An agent cannot complete this alone — stop and ask the operator for the code. |
| `DELETE vpcs/{vpcId}` | No body needed — the id is taken from the path (the spec's `vpc_id` body field is a generation artifact). `vpcId` must be an integer or you get `422`. Success is `204`; if the server declines the delete (e.g. members still attached) it answers `400` with the literal body `{}` — no `id`, no `message` — so treat an empty-object 400 as "VPC still in use". |

## Errors

Most failures arrive as a JSON object carrying a machine-readable `id`, a
human-readable `message`, and usually a numeric `code`:

```json
{ "id": "unprocessable_entity", "message": "…", "code": 50039 }
```

Branch on `id` plus the HTTP status, never on `message` and never on the body's
`code`. That `code` is not the HTTP status: sometimes it is a domain error
number (`50039`), and sometimes it is a different HTTP-looking number than the
response actually carried — an invalid SSH key answers `403` with `"code": 400`
in the body. The full set of `id` values is:

| `id` | HTTP | What it means for you |
| --- | --- | --- |
| `unprocessable_entity` | 422 | Input the server rejected. Fix the payload; retrying it unchanged almost never helps. The exceptions are the 422s that report a transient state rather than bad input — a locked VM, or a quota that has to free up |
| `not_found` | 404 | The resource does not exist, or is not yours — ownership failures are deliberately indistinguishable from absence |
| `unauthorized` | 401 | A good token from an unregistered IP, or an authorization failure inside the request (an ability or policy check) — see rule 1. A bad or missing token does **not** produce this `id`; it produces the framework envelope described below. Neither does touching someone else's resource: that is a `404` |
| `payment_required` | 402 | Insufficient wallet balance. Retrying never helps; the operator must top up |
| `feature_is_not_active` | 403 | The feature is not enabled for this account. Retrying never helps |
| `server_error` | 500 **or 403** | With HTTP 500: a server-side fault, worth retrying with backoff. With HTTP **403**: a business-rule refusal that reuses this `id` (invalid SSH public key, private networking not enabled on the node, root-password change refused, and other legacy service errors). Never retry a 403 `server_error`; read `message` and fix the input or stop |

Three caveats before you write a parser.

- A second envelope exists with no `id` at all:
  `{"success": false, "code": <http>, "error_code": <int>, "message": "..."}`.
  It is what you get whenever the failure happens before or outside the public
  API's own error mapping: a missing, malformed, expired or revoked token
  (`401`, `error_code` 30002), a token without the required ability (`401`,
  30002), a generic framework HTTP error such as a route-level 404 (`error_code`
  30006), or a framework 403 (30003). Handle both shapes; key on `id` when
  present, else on `code`/`error_code`.
- `DELETE vpcs/{vpcId}` can answer `400` with the literal body `{}` — neither
  shape. See gotchas.
- An error is not guaranteed to be JSON: a misrouted or unauthenticated
  request can return an HTML page, so check the content type before parsing.

`payment_required` and `feature_is_not_active` are **not** in the spec, which
documents no 402 or 403 responses anywhere. They are nonetheless the most
likely way a `POST vms` fails for an account that is otherwise set up
correctly.

## Where the spec is wrong

`server-api.openapi` is generated from the server code, so its paths, methods
and enums are reliable. These specific items are not — they have been checked
against the running server, and this file overrides the spec for them:

| Spec says | Reality |
| --- | --- |
| `action.status` example is `success` | Never sent. The enum in the same block is right: `in-progress`, `completed`, `errored` |
| `POST vms` description says `202 Accepted` | Returns `201`. The documented response schema is the correct one |
| Firewall rule `ports` response description advertises ranges (`"8000-9000"`) | Not writable. The request schema's `number` type is what the server enforces — one port per rule |
| Several `GET` operations carry a `requestBody` (e.g. `vms/{id}/actions/{actionId}`) | Artifact of code generation. Send no body on `GET` |
| No `402` or `403` responses documented | Both occur — see "Errors" |
| `securitySchemes` describes only how to obtain a token | Tokens are additionally restricted by IP — see rule 1 |
| `DELETE vpcs/{vpcId}` takes a `vpc_id` body field | Ignored; the path parameter is used. Send no body |
| `GET reserved_ips/{ip}/actions/{actionId}` documents only a `404` "route could not be found" | The route exists. It returns `200` with a fabricated `completed` action for any id — see "Reserved IPs" |
| `enable_ipv6` looks like the other asynchronous actions | It completes inside the request and returns the synthetic `id: 0` action — see "VM actions" |
| Region slugs are mixed-case in every example (`Tehran2`) | The server lowercases them. Compare case-insensitively |
| `POST vms` `image` accepts "the image ID … or the unique slug" | A number is read as a **backup id**, not an image id. Send the slug — see gotchas |

Success codes also vary more than the spec's prose suggests: `POST vms`,
`POST vms/{id}/actions` and `POST account/keys` are `201`, `POST firewalls` and
`POST reserved_ips` are `202`, `POST reserved_ips/{ip}/actions` and
`PUT firewalls/{id}` are `200`, and every `DELETE` plus the firewall rule and
attachment endpoints are `204` with an empty body. Treat any 2xx as success
rather than matching an exact code.

If you discover a new field, error shape, or behavior while integrating, extend
`server-api.openapi` and this file rather than keeping private undocumented
knowledge.
