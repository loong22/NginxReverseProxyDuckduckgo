# NginxReverseProxyDuckduckgo
Use nginx to reverse proxy the duckduckgo search engine

The nginx configuration file is located in the sites-enabled folder. You can choose either example.com or search.example.com; one is for root domain reverse proxy, and the other is for subdomain reverse proxy. Note: For root domain reverse proxy, you can directly open `https://example.com` in your browser to search. For subdomain reverse proxy, you must access `https://search.example.com/?q=1` to search. Choose the configuration file that best suits your needs.

## DNS
This repository contains an Nginx reverse-proxy configuration that proxies DuckDuckGo search and related assets through the following hostnames:
- A search.example.com/example.com  x.x.x.x(Your IPV4 Address)
- A external-content.example.com    x.x.x.x(Your IPV4 Address)
- A links.example.com               x.x.x.x(Your IPV4 Address)
- A improving.example.com           x.x.x.x(Your IPV4 Address)

The proxy adapts upstream links and breaks Subresource Integrity (SRI) checks so resources load correctly under the proxied hostnames.

## Debian 12 — Install and deploy (For search.example.com)

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

