# NginxReverseProxyDuckduckgo
Use nginx to reverse proxy the duckduckgo search engine

## Overview
This repository contains an Nginx reverse-proxy configuration that proxies DuckDuckGo search and related assets through the following hostnames:
- search.example.com
- external-content.example.com
- links.example.com
- improving.example.com

The proxy adapts upstream links and breaks Subresource Integrity (SRI) checks so resources load correctly under the proxied hostnames.

## Prerequisites
- A public server with a reachable IPv4/IPv6 address.
- Nginx compiled with the following modules available:
  - http_ssl_module
  - http_proxy_module
  - http_sub_module (for sub_filter)
  - http_headers_module
- Ports 80 and 443 reachable from the Internet (or DNS validation for certificates).

## DNS
Create DNS records for each hostname pointing to your server IP(s). Examples:
- A     search.example.com       -> your_server_ipv4
- AAAA  search.example.com       -> your_server_ipv6  (optional)
Repeat for:
- external-content.example.com
- links.example.com
- improving.example.com

Alternatively, use CNAMEs if appropriate for your DNS setup.

## SSL certificates (recommended: acme.sh / Let's Encrypt ECDSA)
Example steps using acme.sh with standalone mode (requires port 80 free):

1. Install acme.sh (if not already):
   curl https://get.acme.sh | sh

2. Issue ECDSA certificates (example for search.example.com):
   acme.sh --issue --standalone -d search.example.com --keylength ec-256

3. Install certificate to the paths used by the nginx config:
   acme.sh --installcert -d search.example.com \
     --ecc \
     --key-file /root/.acme.sh/search.example.com_ecc/search.example.com.key \
     --fullchain-file /root/.acme.sh/search.example.com_ecc/fullchain.cer \
     --reloadcmd "systemctl reload nginx"

Repeat issue/install for:
- external-content.example.com
- links.example.com
- improving.example.com

If port 80/443 cannot be used for HTTP validation, use DNS validation with your DNS provider's API:
   acme.sh --issue --dns dns_provider -d example.com ...

Adjust the --keylength/--ecc options per your desired key type.

## Firewall
Allow incoming traffic on:
- TCP 80  (HTTP, needed for ACME standalone HTTP validation and redirects)
- TCP 443 (HTTPS)

Example (ufw):
   ufw allow 80/tcp
   ufw allow 443/tcp

## Nginx notes and testing
- The config uses sub_filter to rewrite upstream domains and to disable integrity attributes:
  - sub_filter 'integrity=' 'no-integrity='
  - sub_filter replaces duckduckgo.com → example.com and specific subdomains to your proxied hostnames.
- Ensure nginx has http_sub_module (sub_filter) available.
- Test configuration:
   nginx -t
- Reload after certificate install or config changes:
   systemctl reload nginx

## Security and behavior notes
- The configuration intentionally breaks SRI (to avoid integrity mismatch) and rewrites origins/cookies to the proxied hostnames.
- The configuration hides upstream Content-Security-Policy and X-Frame-Options headers and sets a permissive Access-Control-Allow-Origin for specific endpoints. Review these settings if you require stricter policies.
- The user-agent check blocks common crawlers; adjust or remove as appropriate for your use case.

## Troubleshooting
- If assets fail to load, verify sub_filter types include the content MIME type and that proxy_set_header Accept-Encoding is set to allow sub_filter to operate.
- If ACME fails in standalone mode because another process listens on port 80, either stop that process temporarily or use DNS validation.
- Verify certificate file paths in the nginx config match the files produced by your ACME tool.

## Debian 12 — Install and deploy (example)

1. Update system and install Nginx
   - Update packages:
     sudo apt update && sudo apt upgrade -y
   - Install nginx:
     sudo apt install -y nginx

2. Verify nginx has required modules
   - Check compiled modules:
     nginx -V 2>&1 | grep -- '--with-http_sub_module\|--with-http_ssl_module\|--with-http_stub_status_module'
   - If http_sub_module is missing, install a package that includes it (e.g. nginx-extras on some distributions), use the official nginx.org package, or compile with --with-http_sub_module.

3. Deploy nginx config
   - Place your config file:
     sudo mkdir -p /etc/nginx/sites-available /etc/nginx/sites-enabled
     sudo cp /path/to/your/local/search.example.com /etc/nginx/sites-available/search.example.com
     sudo ln -s /etc/nginx/sites-available/search.example.com /etc/nginx/sites-enabled/search.example.com
   - Test and reload:
     sudo nginx -t
     sudo systemctl reload nginx

4. Obtain TLS certificates with acme.sh (example using standalone mode)
   - Install acme.sh:
     curl https://get.acme.sh | sh
     source ~/.bashrc
   - (If nginx listens on port 80, stop it temporarily or use --webroot)
     sudo systemctl stop nginx
   - Issue ECDSA cert:
     acme.sh --issue --standalone -d search.example.com --keylength ec-256
   - Install cert to paths matching the nginx config:
     acme.sh --installcert -d search.example.com \
       --ecc \
       --key-file /root/.acme.sh/search.example.com_ecc/search.example.com.key \
       --fullchain-file /root/.acme.sh/search.example.com_ecc/fullchain.cer \
       --reloadcmd "systemctl reload nginx"
   - Restart nginx if stopped:
     sudo systemctl start nginx
   - Repeat issue/install for external-content.example.com, links.example.com, improving.example.com
   - If you cannot free port 80, use --webroot or DNS challenge:
     acme.sh --issue --webroot /var/www/html -d search.example.com
     or
     acme.sh --issue --dns dns_provider -d search.example.com

5. Firewall and systemd
   - Allow HTTP/HTTPS:
     sudo ufw allow 80/tcp
     sudo ufw allow 443/tcp
     sudo ufw enable
   - Enable nginx on boot:
     sudo systemctl enable nginx

6. Verify
   - Confirm nginx serving and TLS:
     sudo nginx -t
     curl -Iv https://search.example.com
   - Check sub_filter behavior: request a known asset and confirm domains and integrity attributes are rewritten.

7. Browser verification
   - Open a browser and visit the search URL with a query parameter, for example:
     https://search.example.com/?q=1
   - Note: The search requires the q parameter (here q=1 is the search keyword). Do not expect a search to run by visiting https://search.example.com without the query parameter.

8. Notes
   - Keep certificate file paths in nginx config identical to acme.sh install paths.
   - acme.sh installs a cron job for renewals by default; ensure reload command works on renew.
   - For production, review CSP/Access-Control changes and UA blocker rules to match your policy.

