# FaultScope site — deployment guide (VPS)

Public brand: **FaultScope**. Domain: **faultscope.cn** (A records already point at VPS `43.153.215.79`).
Target: Ubuntu 24.04 VPS, user `ubuntu` with sudo, nginx already installed and listening on port 80.

## 0. Fill in BEFORE deploying

| Placeholder | Value | Status |
|---|---|---|
| `support@faultscope.cn` | support email shown on every page | **FILLED** — confirmed working 2026-11-19 (receiving test passed, DKIM verified). |
| `洛阳市西工区道源网络销售商行` | exact seller legal name | **FILLED** — confirmed by the user 2026-11-19 (business license name). |
| Support phone | `17538802479` | Already filled in everywhere. |

All placeholders are filled (verified 2026-11-19: zero `[SUPPORT_EMAIL]` / `[BUSINESS_NAME]` remain). Skip to step 1.

## 1. Copy the site files to the VPS

From your PC (PowerShell), upload the zip (all 7 files) to the VPS:

```powershell
scp faultscope-site-20261119.zip ubuntu@43.153.215.79:/tmp/
```

Then on the VPS:

```bash
sudo mkdir -p /var/www/faultscope
cd /tmp && unzip -o faultscope-site-20261119.zip
sudo cp /tmp/faultscope-site/*.html /var/www/faultscope/
sudo chown -R www-data:www-data /var/www/faultscope
sudo chmod -R 755 /var/www/faultscope
ls -l /var/www/faultscope
```

## 2. Install the nginx site config

```bash
sudo cp /tmp/faultscope-site/nginx-faultscope.conf /etc/nginx/sites-available/faultscope
sudo ln -s /etc/nginx/sites-available/faultscope /etc/nginx/sites-enabled/faultscope
sudo nginx -t
sudo systemctl reload nginx
```

Note: if `sites-enabled/default` also claims port 80, the `server_name` in this config takes precedence for faultscope.cn, so the default site can stay. If `nginx -t` complains about a duplicate default server, remove the default symlink (`sudo rm /etc/nginx/sites-enabled/default`) and reload.

## 3. HTTPS with certbot

Prerequisites: DNS A records for `@` and `www` resolve to `43.153.215.79` (check with `nslookup faultscope.cn`), and the domain real-name verification has passed (until then DNS will not resolve at all).

```bash
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d faultscope.cn -d www.faultscope.cn
```

Accept the prompts (email for renewal notices; agree to the terms). Certbot edits the nginx config to add the 443 server block and a HTTP→HTTPS redirect.

## 4. Verify

```bash
curl -sI https://faultscope.cn | head -5        # expect: HTTP/2 200
curl -sI https://www.faultscope.cn | head -5    # expect: HTTP/2 200
curl -s https://faultscope.cn/zh | grep -o "<title>[^<]*</title>"
curl -s https://faultscope.cn/terms.html | grep -io "merchant of record" | head -1
curl -s https://faultscope.cn/refund.html | grep -io "30-day" | head -1
curl -s https://faultscope.cn/privacy.html | grep -o "<title>[^<]*</title>"
```

Then open in a browser and check: https://faultscope.cn (English), https://faultscope.cn/zh (Chinese), and the Terms / Refund / Privacy footer links on every page.

## Paddle domain-review checklist — what this site satisfies

| Paddle requirement | Status |
|---|---|
| Live site over HTTPS | After step 3 |
| Product description on the site | `index.html` hero + "How it works" |
| Pricing visible | `index.html` + `zh.html` pricing section (¥2,000/yr software subscription, CNY; hardware + install arranged separately, not sold online) |
| Terms of Service linked from navigation/footer | `terms.html`, linked on every page |
| Refund Policy linked from navigation/footer | `refund.html`, linked on every page; 30-day window |
| Privacy Policy linked from navigation/footer | `privacy.html`, linked on every page |
| Terms name Paddle as merchant of record | Terms §1 + footer on every page |
| Support email + phone reachable | Phone + support@faultscope.cn filled in (verified 2026-11-19) |
| Seller (sole-trader) name in Terms | Filled in: 洛阳市西工区道源网络销售商行 |

## Troubleshooting

- **Domain does not resolve / `curl` fails:** real-name verification (实名审核) not yet passed — check the registrar console. DNS only works after it passes.
- **`nginx -t` fails:** check for a conflicting `default_server` on port 80 (see step 2 note).
- **Certbot fails with a connection error:** port 80 must be reachable from the internet (check the VPS firewall/security group allows 80 and 443) and DNS must have propagated (wait up to ~10 min after adding records; TTL is 600s).
- **Certbot renewal:** `certbot renew` runs automatically via a systemd timer; check with `sudo systemctl status certbot.timer`.

## Later (not part of this deployment)

- Paddle checkout links: once the Paddle account and products are created, replace the "Order — contact support" mailto button with the Paddle checkout link.
- The diagnosis service itself (rust-core / py-core) is a separate deployment; this site is static only.
