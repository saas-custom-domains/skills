---
name: saas-custom-domains
description: Add custom domains with automatic SSL to a multi-tenant SaaS app using the SaaS Custom Domains API. Use when a user wants their customers to serve the app on their own domain (app.customer.com), needs to create or remove custom domains via API, show DNS instructions, verify requests forwarded by the SaaS Custom Domains proxy, or handle its webhooks.
---

# Add custom domains with SaaS Custom Domains

SaaS Custom Domains is a managed reverse proxy. Your user's customer points their domain at it, it issues and renews the TLS certificate, and it forwards every request to the app (the "upstream") with the customer's domain in the `X-Served-For` header. The app never handles certificates or DNS.

- API base URL: `https://app.saascustomdomains.com/api/v1`
- Auth: `Authorization: Bearer <API token>` (tokens: https://app.saascustomdomains.com/api_tokens). Keep it server-side.
- JSON request and response bodies. Rate limit: 360 requests per minute per token.
- Full reference: https://docs.saascustomdomains.com/llms-full.txt and https://docs.saascustomdomains.com/openapi.json
- To manage domains from an MCP client instead of writing API calls (for example, adding one customer's domain or checking why it isn't live), use the MCP server at `https://mcp.saascustomdomains.com/mcp`, signing in with OAuth or using the same token: https://docs.saascustomdomains.com/mcp. Code in the app itself should call the API.

## 1. One-time setup: the upstream

The upstream is the app the custom domains forward to, for example `app.yourapp.com`.

1. `GET /accounts` to find the `account_uuid`.
2. Create the upstream once, in the dashboard or with `POST /accounts/{account_uuid}/upstreams` and `{"host": "app.yourapp.com"}` (defaults: port 443, TLS on). For an upstream that only serves plain HTTP, send `"tls": false, "port": 80`.
3. Save the upstream's `uuid` and `auth_token` as configuration (for example `SCD_UPSTREAM_UUID` and `PROXY_AUTH_TOKEN`). Don't create an upstream per customer.

## 2. Add a customer's domain

`POST /accounts/{account_uuid}/upstreams/{upstream_uuid}/custom_domains`

```json
{ "host": "app.customer.com" }
```

Optional fields:
- `redirect_to_www`: for a root domain (`customer.com`), also serve `www.customer.com` and redirect the root to it. Counts as two domains on the bill.
- `challenge_type`: `http01` (default) or `dns01`. Wildcard domains (`*.customer.com`) must use `dns01`. Use `dns01` when the customer keeps Cloudflare's proxy on.
- `instructions_recipient`: an email address to send step-by-step DNS instructions to (needs a confirmed email sender signature in the dashboard).

Store the returned `uuid`, `host` and `status` with the tenant. Normalize the host first (trim, lowercase, drop a trailing dot), and reject your own app's hostnames so a tenant can't claim them. Every created domain is billed, whether or not DNS is set up, so delete abandoned ones.

## 3. Tell the customer which DNS records to add

`GET .../custom_domains/{uuid}/dns_records` returns the exact records for a domain (`type`, `name`, `value`, `purpose`), the same ones the dashboard shows. The rules behind them:

- Subdomain (`app.customer.com`): CNAME `app.customer.com` → `in.saascustomdomains.com`. If the account has a custom CNAME target (for example `in.yourapp.com`), show that instead.
- Root domain (`customer.com`): two A records, `75.2.96.173` and `99.83.186.151`. Both are required. No AAAA records.
- With `redirect_to_www`: the same record for `www.customer.com` too.
- `dns01` (and every wildcard): also a CNAME from `delegated_domain_control_validation_record_hostname` to `delegated_domain_control_validation_record_value` (both in the domain response). It must stay in place and DNS only, because renewals use it.
- Cloudflare users: set the records to DNS only (grey cloud) unless the domain uses `dns01`.

## 4. Know when the domain is live

The domain is ready when `status` is `dns_records_found` and `tls_certificate_issued` is `true`. With `http01`, the certificate is issued on the first request after DNS is found.

- Webhooks (preferred): add an endpoint in the dashboard and handle `domain_dns_proxy_record_valid`, `domain_dns_proxy_record_invalid`, `tls_certificate_issued` and `domain_deleted`. Match by `domain.uuid`. Verify the webhook token (sent as the `token` query parameter and the `X-SCD-Webhook-Token` header) with a constant-time compare, and reject the request if your expected token is empty.
- Polling: `GET .../custom_domains/{uuid}`. DNS is re-checked automatically (often in the first minutes, then every 3 hours). `POST .../custom_domains/{uuid}/verify_dns_records` forces a check but is limited to 5 requests per minute.

## 5. Serve requests that arrive through the proxy

Every forwarded request has:
- `Host` and `X-Forwarded-Host`: the upstream's host (`app.yourapp.com`), not the customer's domain.
- `X-Served-For`: the customer's domain. Use it to find the tenant.
- `X-SaaS-Domains-Auth-Token`: the upstream's auth token.
- `X-Forwarded-Proto: https`, `X-Forwarded-For` and `X-SaaS-Domains-IP`: the visitor's IP.

Do this before any tenant lookup:
1. If `X-Served-For` is present, require `X-SaaS-Domains-Auth-Token` to equal `PROXY_AUTH_TOKEN` (constant-time compare; reject if the expected token is empty). Otherwise anyone who can reach the upstream could fake `X-Served-For`.
2. Look up the tenant by the normalized `X-Served-For` host. Unknown or inactive domains get a 404.
3. Use the customer's domain, not the upstream host, when building absolute URLs, redirects and canonical links.

Framework gotchas:
- CSRF and origin checks compare the browser's `Origin` (the customer's domain) with the request host (the upstream). After the token check, make the framework see the customer's domain (for example by setting the request host from `X-Served-For`).
- If a CDN or cache sits in front of the upstream, add `X-Served-For` to its cache key, or one customer's cached page can be served on another customer's domain.
- Our proxy IPs change. Don't allowlist IPs; check the auth token. If a WAF sits in front of the upstream, let requests with the correct auth token through.

## 6. Change or remove a domain

- Remove: `DELETE .../custom_domains/{uuid}`.
- There is no update endpoint. To change settings or move a domain to another upstream, delete it and create it again right away. If the DNS records stay the same, traffic keeps working.

## Test locally without an account

Run the mock API and proxy: `docker run --rm -p 4010:4010 -p 4080:4080 -e SCD_MOCK_UPSTREAM_HOST=host.docker.internal ghcr.io/saas-custom-domains/scd-mock:latest`. API at `http://localhost:4010/api/v1` with token `scd_test_token`, account `acc_local`, upstream `upstream_local`, proxy auth token `local_proxy_token`. Domains ending in `.localhost` go live after 2 seconds, and `http://acme.localhost:4080/` reaches your app through the mock proxy. Details: https://github.com/saas-custom-domains/scd-mock

## Examples

Runnable examples with guides for Next.js, Nuxt, SvelteKit, Express, Hono, Rails, Django, FastAPI, Laravel, Go, Phoenix, ASP.NET Core and Spring Boot: https://github.com/saas-custom-domains
