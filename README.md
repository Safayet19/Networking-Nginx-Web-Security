
<h1 align="center">Networking, Nginx & Web Security</h1>

<p align="center">A practical guide to networking commands, Nginx web servers, reverse proxies, DNS, HTTPS, firewalls, and troubleshooting.</p>

---

This reference combines **essential networking, Nginx, and web-server security** in one place. Commands have complete examples; configuration blocks are ready to adapt. The hands-on lab runs a small Node.js backend behind Nginx, with optional public DNS and HTTPS.

> **Environment:** Ubuntu Server 22.04/24.04 with `sudo` privileges and an internet connection. Run Bash commands on the **Ubuntu server**, not in Windows Git Bash unless you are connected to that server over SSH. Other Linux distributions can use different package names and configuration paths. Use a test VM/server first.
>
> **Placeholder rule:** `app.your-real-domain.com` means a domain you actually control; `example.com` is used only for public networking tests. **Do not request a TLS certificate for the literal example domain.** Never expose SSH or application ports without checking your firewall rules.

## Table of Contents

1. [Networking Fundamentals](#1-networking-fundamentals)
2. [Essential Networking Commands](#2-essential-networking-commands)
3. [Nginx Installation and Management](#3-nginx-installation-and-management)
4. [Nginx Configuration Basics](#4-nginx-configuration-basics)
5. [Host a Static Website](#5-host-a-static-website)
6. [Nginx Reverse Proxy](#6-nginx-reverse-proxy)
7. [Domain and DNS Configuration](#7-domain-and-dns-configuration)
8. [HTTPS and TLS Certificates](#8-https-and-tls-certificates)
9. [Firewall and Web Security](#9-firewall-and-web-security)
10. [Logs and Troubleshooting](#10-logs-and-troubleshooting)
11. [Practical Project: Secure Reverse Proxy](#11-practical-project-secure-reverse-proxy)

---

## 1. Networking Fundamentals

| Concept | Meaning | Practical example |
|---|---|---|
| IP address | Address assigned to a network interface | `192.168.1.10` (private IPv4) |
| Public IP | Internet-routable address for a server or gateway | The address in your DNS `A` record |
| CIDR | Network range notation | `192.168.1.0/24` contains 256 IPv4 addresses, not necessarily 256 usable hosts |
| Gateway | Router used to reach other networks | The default route shown by `ip route` |
| DNS | Translates domain names into addresses | `A` record: domain to IPv4; `AAAA`: domain to IPv6 |
| TCP | Connection-oriented transport | HTTPS normally uses TCP port 443; HTTP/3 uses QUIC over UDP |
| UDP | Datagram transport | Many DNS queries and QUIC connections |
| Port | Identifies an application endpoint on a host | SSH `22`, HTTP `80`, HTTPS `443` |
| HTTP | Web application protocol | `http://localhost:3000` |
| HTTPS | HTTP secured with TLS | `https://your-domain.com` |
| Reverse proxy | Receives requests, forwards them to a backend | Nginx `:443` -> Node `127.0.0.1:3000` |
| Firewall | Controls permitted network traffic | Allow TCP 80/443 and restrict SSH |

**Request path:** Browser -> DNS lookup -> server IP and TCP/TLS connection -> Nginx -> backend service -> Nginx -> browser. DNS does **not** choose the backend port; Nginx does that through `proxy_pass`.

## 2. Essential Networking Commands

The commands using `dig`, `traceroute`, and `nc` require the matching packages, installed in Section 3. These checks do not modify network configuration.

| Task | Complete command | Explanation |
|---|---|---|
| List IP addresses | `ip -br addr` | `-br` shows a concise list of interfaces and assigned addresses. |
| Show routes | `ip route` | Displays the routing table, including the default gateway. |
| Show default route | `ip route show default` | Displays where traffic to other networks normally goes. |
| Check connectivity | `ping -c 4 1.1.1.1` | Sends four ICMP echo requests; blocked ICMP does not always mean the internet is unavailable. |
| Resolve IPv4 DNS | `dig +short A example.com` | Prints IPv4 `A` records without full DNS diagnostics. |
| Resolve IPv6 DNS | `dig +short AAAA example.com` | Prints IPv6 `AAAA` records if present. |
| View detailed DNS answer | `dig example.com` | Includes response flags, answer, and DNS server information. |
| Alternative system resolver | `getent ahosts example.com` | Resolves the hostname using the system's name-service configuration. |
| Show listening TCP sockets | `sudo ss -ltnp` | `-l` listening, `-t` TCP, `-n` numeric, `-p` processes. |
| Show listening TCP and UDP | `sudo ss -tulnp` | Adds UDP sockets with `-u`; shows listening endpoints and processes. |
| Check backend listener | `sudo ss -ltnp '( sport = :3000 )'` | Shows whether a process listens on port 3000 and on which address. |
| Test HTTP locally | `curl -i http://127.0.0.1:3000/` | `-i` displays response headers together with the body. |
| Check website headers | `curl -I https://example.com` | `-I` requests headers only. |
| Follow redirects | `curl -IL https://example.com` | `-L` follows redirects; `-I` prints headers. |
| Show HTTP request details | `curl -v http://127.0.0.1:3000/` | `-v` shows connection and request/response diagnostics. |
| Check a TCP port | `nc -vz example.com 443` | `-z` checks connection without sending data; `-v` prints result. |
| Trace network route | `traceroute example.com` | Shows attempted network hops; routers may block replies. |
| Inspect a public TLS handshake | `openssl s_client -connect example.com:443 -servername example.com </dev/null` | `-connect` selects the server/port; `-servername` sends the TLS hostname (SNI). |

**To display the public certificate expiration dates:**

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null | \
  openssl x509 -noout -dates
```

## 3. Nginx Installation and Management

On an Ubuntu server, install the tools used in this guide:

```bash
sudo apt update
sudo apt install -y nginx curl dnsutils iproute2 openssl ufw netcat-openbsd traceroute nodejs
nginx -v
node --version
sudo systemctl enable --now nginx
sudo nginx -t
```

`nginx -t` checks the configuration **before** you reload it. On a different Linux distribution, the service name, package manager, and site configuration layout can differ.

| Task | Complete command | Explanation |
|---|---|---|
| Check Nginx version | `nginx -v` | Prints the installed version. |
| Verify Nginx configuration | `sudo nginx -t` | Detects syntax errors or missing referenced files. |
| Start Nginx | `sudo systemctl start nginx` | Starts the server immediately. |
| Stop Nginx | `sudo systemctl stop nginx` | Stops serving new traffic. |
| Restart Nginx | `sudo systemctl restart nginx` | Restarts the service; briefly interrupts connections. |
| Reload configuration | `sudo systemctl reload nginx` | Applies valid changes with minimal disruption. |
| Show service status | `sudo systemctl status nginx --no-pager` | Shows whether Nginx is active. |
| Enable on boot | `sudo systemctl enable nginx` | Starts Nginx after reboot. |
| Inspect service errors | `sudo journalctl -u nginx -n 50 --no-pager` | Displays recent Nginx service events. |

**Recommended change sequence:** Edit configuration -> `sudo nginx -t` -> `sudo systemctl reload nginx`. Do not reload an invalid configuration.

## 4. Nginx Configuration Basics

Ubuntu's packaged Nginx normally loads site definitions from `/etc/nginx/sites-enabled/`, which commonly contains symlinks to files in `/etc/nginx/sites-available/`.

| Path / directive | Purpose | Example |
|---|---|---|
| `/etc/nginx/nginx.conf` | Main Nginx configuration | Includes the site's server blocks |
| `/etc/nginx/sites-available/` | Stores site configurations | `proxy-demo` |
| `/etc/nginx/sites-enabled/` | Selects enabled site files | Symlink to `proxy-demo` |
| `server { ... }` | Defines a virtual host | HTTP server on port 80 |
| `listen` | Address and port Nginx accepts | `listen 80;` |
| `server_name` | HTTP hostname to match | `server_name app.example.com;` (illustration only) |
| `root` | Directory of static files | `root /var/www/site-demo;` |
| `index` | Default directory index | `index index.html;` |
| `location` | Chooses how to handle URL paths | `location / { ... }` |
| `try_files` | Checks filesystem paths | `try_files $uri $uri/ =404;` |
| `proxy_pass` | Sends requests to a backend | `proxy_pass http://127.0.0.1:3000;` |
| `proxy_set_header` | Controls forwarded request headers | `proxy_set_header Host $host;` |

**View actual active configuration:**

```bash
sudo nginx -T
```

This prints all loaded configuration. It may expose paths or sensitive settings, so do not paste its full output publicly without reviewing it.

## 5. Host a Static Website

This independent local exercise uses port **8081 on `127.0.0.1`**. It will not conflict with the reverse-proxy example on port 80 and is not publicly exposed.

**Step 1 — Create a page:**

```bash
sudo mkdir -p /var/www/site-demo
printf '<!doctype html><html><body><h1>Hello from Nginx</h1></body></html>\n' | sudo tee /var/www/site-demo/index.html > /dev/null
```

**Step 2 — Create `/etc/nginx/sites-available/site-demo`:**

```nginx
server {
    listen 127.0.0.1:8081;
    server_name localhost;

    root /var/www/site-demo;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Use `sudo nano /etc/nginx/sites-available/site-demo` to save the configuration above. Then enable, check, and test it:

```bash
sudo ln -s /etc/nginx/sites-available/site-demo /etc/nginx/sites-enabled/site-demo
sudo nginx -t && sudo systemctl reload nginx
curl http://127.0.0.1:8081/
```

**Expected:** HTML containing `Hello from Nginx`. Create the symlink only once; if it already exists, inspect it instead of duplicating it.

**Optional cleanup of this static-only exercise:**

```bash
sudo rm /etc/nginx/sites-enabled/site-demo
sudo nginx -t && sudo systemctl reload nginx
sudo rm /etc/nginx/sites-available/site-demo
sudo rm -r /var/www/site-demo
```

The cleanup deletes **only the files created for this test**. Do not run it if you have repurposed those paths for a real site.

## 6. Nginx Reverse Proxy

Nginx can accept web requests on port **80/443** and send them to an application bound only to **`127.0.0.1:3000`**. The backend is not directly reachable from other machines. The full runnable backend and installation steps appear in Section 11.

**Nginx proxy configuration (`/etc/nginx/sites-available/proxy-demo`):**

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

| Directive | Exact effect |
|---|---|
| `listen 80` | Accepts HTTP on port 80. |
| `server_name localhost` | Selects this server block for HTTP requests whose Host is `localhost` (change for a real domain). |
| `location /` | Matches paths under `/`. |
| `proxy_pass http://127.0.0.1:3000` | Sends requests to the locally running backend, preserving the request URI in this form. |
| `proxy_set_header Host $host` | Passes the requested hostname to the backend. |
| `X-Real-IP` | Passes the client IP seen by Nginx. |
| `X-Forwarded-For` | Appends the client IP to a forwarding chain; application code should only trust it behind known proxies. |
| `X-Forwarded-Proto` | Communicates the original HTTP or HTTPS scheme. |

**Spring Boot alternative:** If a Spring Boot app is already listening on local port 8080, set in its `application.properties`:

```properties
server.address=127.0.0.1
server.port=8080
server.forward-headers-strategy=framework
```

Then change the single `proxy_pass` line to `proxy_pass http://127.0.0.1:8080;`. Do **not** use both backend ports in the same example unless you actually have two services and designed separate routes. Restart the Spring Boot application after changing its properties.

## 7. Domain and DNS Configuration

This section is **optional for local practice**. A real public HTTPS deployment needs a domain you control, a publicly reachable server, correct DNS, and inbound TCP ports 80/443 permitted by both the host firewall and any cloud firewall/security group.

| DNS setting | Example | Meaning |
|---|---|---|
| `A` | Host `app` -> your server's public IPv4 | Maps a subdomain to an IPv4 address. |
| `AAAA` | Host `app` -> public IPv6 (only if configured) | Maps to IPv6; omit an incorrect record. |
| `CNAME` | Host `www` -> another hostname | Alias to another domain name; not an IP address. |
| TTL | `300` seconds (if your provider permits) | DNS caching time; updates may take time to propagate. |

**At your DNS provider:** Add an `A` record for your chosen host (for example, `app`) pointing to your actual server public IP.

**Set your own domain (replace the placeholder):**

```bash
export DOMAIN=app.your-real-domain.com
# Run only after Section 11 created /etc/nginx/sites-available/proxy-demo:
sudo sed -i "s/server_name localhost;/server_name $DOMAIN;/" /etc/nginx/sites-available/proxy-demo
sudo nginx -t && sudo systemctl reload nginx

dig +short A "$DOMAIN"
dig +short AAAA "$DOMAIN"
curl -I "http://$DOMAIN/"
```

**Expected:** `dig +short A` returns the **actual server public IPv4**; HTTP reaches your Nginx site once firewall rules permit it. If `AAAA` exists, IPv6 must be configured and working too. If a default website appears, inspect the `server_name`, active configuration, request Host header, and DNS result.

## 8. HTTPS and TLS Certificates

**Only for a publicly reachable server with a real domain.** Let's Encrypt cannot validate a local-only hostname such as `localhost`. The domain must resolve to this server, and port 80 normally needs to be reachable for HTTP validation. Back up important Nginx settings before allowing Certbot to modify them.

The Ubuntu Server documentation recommends installing Certbot through **snap**:

```bash
sudo apt install -y snapd
sudo snap install --classic certbot
sudo /snap/bin/certbot --nginx -d "$DOMAIN" --redirect
```

The last command requests a certificate for the domain in Section 7, adds TLS configuration to the matching Nginx server block, and configures an HTTP-to-HTTPS redirect when supported. It may prompt for an email address and agreement to the certificate authority's terms. Do not run it with the placeholder domain.

| Task | Complete command | Explanation |
|---|---|---|
| Check HTTPS response | `curl -I "https://$DOMAIN/"` | Tests successful TLS connection and response headers. |
| Inspect issued certificates | `sudo /snap/bin/certbot certificates` | Lists managed certificate names and expiration dates. |
| Test renewal safely | `sudo /snap/bin/certbot renew --dry-run` | Tests renewal without deploying a replacement certificate. |
| Check automatic renewal timer | `sudo systemctl status snap.certbot.renew.timer --no-pager` | Inspect the snap-managed renewal timer (if installed). |
| Verify HTTP redirects to HTTPS | `curl -I "http://$DOMAIN/"` | Should show an HTTP redirect after successful Certbot configuration. |

**TLS basics:** A certificate proves control of the requested hostname and enables encrypted connections. The certificate must match the hostname; pointing DNS to your server alone does not create a certificate. Never commit the files under `/etc/letsencrypt/live/` containing private keys.

## 9. Firewall and Web Security

**Before enabling UFW on a remote server, confirm the actual SSH port and allow it**, or you may lock yourself out. The example assumes SSH uses TCP 22. Cloud security groups / network ACLs can impose additional restrictions beyond UFW.

| Task | Complete command | Explanation |
|---|---|---|
| Inspect firewall | `sudo ufw status verbose` | Show UFW state, defaults, and current rules. |
| Allow SSH on standard port | `sudo ufw allow 22/tcp` | Keep remote SSH accessible **if** your SSH server uses port 22. |
| Allow HTTP | `sudo ufw allow 80/tcp` | Permit public HTTP for Nginx and certificate validation. |
| Allow HTTPS | `sudo ufw allow 443/tcp` | Permit encrypted web traffic. |
| Activate UFW | `sudo ufw enable` | Enable the firewall **after** verifying SSH access rules. |
| Inspect numbered rules | `sudo ufw status numbered` | Show active rules with indexes. |
| Restrict SSH to one trusted IPv4 | `sudo ufw allow from 192.0.2.10 to any port 22 proto tcp` | Example syntax; replace the documentation IP with your actual trusted IP and review existing rules first. |
| Remove a specific test rule | `sudo ufw delete allow 80/tcp` | Deletes the corresponding UFW allow rule; do not remove it while serving HTTP/certificate challenges. |

**What should be exposed:**

- Public: Nginx on **80** (validation/redirect) and **443** (HTTPS).
- Restricted: **22** (SSH) to approved administrators or networks.
- Private: **3000** (Node) or **8080** (Spring Boot); bind the backend to `127.0.0.1` rather than opening these ports to the internet.

Other useful safeguards: keep Ubuntu/Nginx updated, remove unused site configurations, do not place credentials in web-root folders, and trust `X-Forwarded-*` headers only from the intended reverse proxy. A certificate is **not** a substitute for application security.

## 10. Logs and Troubleshooting

| Task | Complete command | Explanation |
|---|---|---|
| Validate Nginx config | `sudo nginx -t` | Find syntax, include-file, or certificate-path errors before reload. |
| Watch Nginx error log | `sudo tail -f /var/log/nginx/error.log` | Stream errors as requests fail; stop with Ctrl+C. |
| Review recent access requests | `sudo tail -n 30 /var/log/nginx/access.log` | Show the latest 30 access-log lines. |
| Review service errors | `sudo journalctl -u nginx -n 50 --no-pager` | Find startup/reload failures. |
| Test backend directly | `curl -i http://127.0.0.1:3000/` | Confirms the application responds independently of Nginx. |
| Test Nginx virtual host locally | `curl -i -H 'Host: localhost' http://127.0.0.1/` | Sends the correct Host header, even if a default Nginx site also exists. |
| Check backend port | `sudo ss -ltnp '( sport = :3000 )'` | Confirms which address and process listens on port 3000. |
| Check Nginx HTTP port | `sudo ss -ltnp '( sport = :80 )'` | Confirms Nginx is listening on port 80. |
| Check name resolution | `dig +short A "$DOMAIN"` | Compare the DNS answer with the actual public server IP. |

### Common failures

| Symptom | Likely reason | Specific fix |
|---|---|---|
| Nginx service will not reload | Syntax error or invalid referenced file | Run `sudo nginx -t`, read the error, correct the named file, then reload. |
| `403 Forbidden` | Missing directory index or filesystem access denied | Check `root`, `index`, and directory permissions; inspect `/var/log/nginx/error.log`. |
| `404 Not Found` | Requested file/path does not exist or the wrong server block matched | Check `root`, `try_files`, `server_name`, and request Host header. |
| `502 Bad Gateway` | Backend unavailable or wrong upstream address/port | Run `curl -i http://127.0.0.1:3000/`; inspect application logs and `proxy_pass`. |
| `504 Gateway Timeout` | Backend takes too long or is unresponsive | Inspect app health, resource use, and timeout/error logs before increasing proxy timeouts. |
| Default Nginx page appears | Wrong Host header or another virtual host selected | Use `curl -H 'Host: localhost' http://127.0.0.1/`; inspect `sudo nginx -T`. |
| Browser cannot reach public HTTP | DNS, cloud firewall, UFW, or Nginx listener misconfigured | Check `dig`, security group, `ufw status`, and `ss -ltnp`. |
| TLS certificate request fails | Domain does not resolve to reachable server or port 80 blocked | Fix DNS and public inbound access; then rerun Certbot. |
| Browser warns about certificate | Expired certificate, hostname mismatch, or incomplete TLS configuration | Check hostname, `certbot certificates`, renewal timer, and server logs. |
| Node app exits | App error or missing runtime | Check `node --version`, `journalctl -u proxy-demo`, and `curl` to backend. |

**Debugging order:** Backend `curl` -> Nginx `nginx -t` -> local Host-header `curl` -> DNS -> public firewall -> HTTPS. Changing random settings before finding the failing layer usually makes troubleshooting harder.

## 11. Practical Project: Secure Reverse Proxy

**Goal:** Run a Node.js HTTP service privately on `127.0.0.1:3000`, proxy it through Nginx on port 80, and optionally connect your domain and enable HTTPS. It uses **no npm dependencies, Docker, database, or external application source code**.

### Step 1 — Install prerequisites

Run the installation commands from Section 3. Confirm:

```bash
nginx -v
node --version
sudo systemctl status nginx --no-pager
```

### Step 2 — Create the backend application

```bash
mkdir -p "$HOME/proxy-demo"
cat > "$HOME/proxy-demo/server.js" <<'JS'
const http = require('node:http');

const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ message: 'Hello through Nginx', path: req.url }));
});

server.listen(3000, '127.0.0.1', () => {
  console.log('Backend listening on http://127.0.0.1:3000');
});
JS
node --check "$HOME/proxy-demo/server.js"
```

### Step 3 — Run the backend as a Linux service

Create one **systemd** unit for the current non-root Ubuntu user. The unquoted `EOF` below intentionally expands your username, home directory, and Node executable path **when the unit is created**.

```bash
sudo tee /etc/systemd/system/proxy-demo.service > /dev/null <<EOF
[Unit]
Description=Local Node backend for Nginx practice
After=network.target

[Service]
Type=simple
User=$(id -un)
WorkingDirectory=$HOME/proxy-demo
ExecStart=$(command -v node) $HOME/proxy-demo/server.js
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now proxy-demo
sudo systemctl status proxy-demo --no-pager
curl -i http://127.0.0.1:3000/
```

**Expected:** JSON such as `{"message":"Hello through Nginx","path":"/"}`. Confirm the app listens **only** on loopback:

```bash
sudo ss -ltnp '( sport = :3000 )'
```

### Step 4 — Configure Nginx as a reverse proxy

This creates the working server block from Section 6. The **quoted** `NGINX` heredoc is important: it prevents Bash from replacing Nginx variables such as `$host` or `$remote_addr`.

```bash
sudo tee /etc/nginx/sites-available/proxy-demo > /dev/null <<'NGINX'
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
NGINX

sudo ln -s /etc/nginx/sites-available/proxy-demo /etc/nginx/sites-enabled/proxy-demo
sudo nginx -t && sudo systemctl reload nginx
curl -i -H 'Host: localhost' http://127.0.0.1/
```

**Expected:** The same JSON body through Nginx, with an HTTP 200 response. We test with an explicit Host header because Ubuntu's default Nginx website may still be present. If the `proxy-demo` symlink already exists, inspect it; do not create a duplicate.

### Step 5 — Check local networking and security

```bash
sudo ss -ltnp '( sport = :80 )'
sudo ss -ltnp '( sport = :3000 )'
sudo ufw status verbose
```

**Expected:** Nginx listens on port 80; Node listens on `127.0.0.1:3000`. On a remote server, allow your actual SSH port before enabling UFW. Use Section 9 to allow HTTP/HTTPS and the cloud firewall/security group as appropriate.

### Step 6 — Optional: Connect your real domain

If you own a domain and your server has a public IP, complete Section 7. Add the DNS `A` record, set `DOMAIN`, update `server_name`, and test:

```bash
# Replace the value before running:
export DOMAIN=app.your-real-domain.com
sudo sed -i "s/server_name localhost;/server_name $DOMAIN;/" /etc/nginx/sites-available/proxy-demo
sudo nginx -t && sudo systemctl reload nginx
dig +short A "$DOMAIN"
curl -i "http://$DOMAIN/"
```

**Expected:** The domain resolves to your public IP and the HTTP request returns the Node JSON. A private VM without public access can complete Steps 1-5 and skip the public steps.

### Step 7 — Optional: Add HTTPS

After domain DNS and inbound ports work, use Section 8:

```bash
sudo apt install -y snapd
sudo snap install --classic certbot
sudo /snap/bin/certbot --nginx -d "$DOMAIN" --redirect
curl -I "https://$DOMAIN/"
sudo /snap/bin/certbot renew --dry-run
```

**Expected:** A valid TLS response on HTTPS and an HTTP-to-HTTPS redirect. Certificate issuance requires a **real, publicly reachable domain** and may fail if a DNS `AAAA` record routes to an unconfigured IPv6 endpoint.

### Step 8 — Verify and clean up only the practice resources

```bash
sudo systemctl status proxy-demo --no-pager
sudo systemctl status nginx --no-pager
sudo journalctl -u proxy-demo -n 20 --no-pager
```

**Optional cleanup for a local-only lab** (do **not** run if you are using the domain/certificate or the site for real traffic):

```bash
sudo systemctl disable --now proxy-demo
sudo rm /etc/systemd/system/proxy-demo.service
sudo systemctl daemon-reload
sudo rm /etc/nginx/sites-enabled/proxy-demo
sudo nginx -t && sudo systemctl reload nginx
sudo rm /etc/nginx/sites-available/proxy-demo
rm -r "$HOME/proxy-demo"
```

Cleanup leaves Nginx installed and does not modify UFW or delete TLS certificates. **If Certbot modified your Nginx configuration, back it up and review certificate and site removal separately before cleanup.**

### Practice Checklist

- [ ] Explain IP, DNS, ports, HTTP/HTTPS, and reverse-proxy request flow.
- [ ] Find IP addresses, listening ports, DNS records, and TLS certificate dates.
- [ ] Install Nginx, validate a configuration, and reload it safely.
- [ ] Host a local static page without making it publicly accessible.
- [ ] Start a private Node backend as a systemd service.
- [ ] Proxy HTTP requests through Nginx and verify the response.
- [ ] If using a public server: configure DNS, UFW/cloud rules, HTTPS, and renewal.
- [ ] Diagnose a stopped backend (`502`), bad server block (`404`), and certificate/DNS issues.
- [ ] Clean up only resources created for practice.

---

## Official References

- [Ubuntu: Configure Nginx](https://ubuntu.com/server/docs/how-to/web-services/configure-nginx/)
- [NGINX: Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy)
- [NGINX: Proxy Module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Ubuntu: Firewall (UFW)](https://documentation.ubuntu.com/server/how-to/security/firewalls/index.html)
- [Ubuntu: Obtain TLS Certificates](https://ubuntu.com/server/docs/how-to/security/obtain-tls-certificates/)
- [Certbot Documentation](https://certbot.eff.org/)

*This README covers core networking and web-server fundamentals. It is not a complete cybersecurity or production hardening standard. Review real DNS, cloud firewall rules, authorization, and application security before exposing a service publicly.*
