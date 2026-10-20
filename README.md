# Enterprise Risk Lab — Secure Web System as an Enterprise Risk Control

Workshop deliverable: a small full-stack system demonstrating enterprise risk controls —
Auth0 authentication, JWT-based authorization at an OAuth 2.0 Resource Server, and HTTPS
with a valid Let's Encrypt certificate behind an Apache reverse proxy.

**Deployed application:** https://danielenterpriseapplication.duckdns.org

| Repository | Contents |
|---|---|
| [securityLab-frontend](https://github.com/A1Daniel1/securityLab-frontend) | React (Vite) SPA with `@auth0/auth0-react` — application code lives in the `frontend/` directory |
| [securityLab-backend](https://github.com/A1Daniel1/securityLab-backend) | Spring Boot 3 / Java 21 OAuth 2.0 Resource Server |

This README is identical in both repositories; evidence images are rendered from the
repository where they were captured.

## Architecture

    Browser
      │ login and access token
      ▼
    Auth0 ───────────────► React frontend ── Bearer token ──► Spring Boot API
                                                              │
    Internet ── Duck DNS ── Apache (httpd) + TLS ────────────┘

A single public origin serves the SPA and proxies `/api/*` to the backend on
`127.0.0.1:8080`. Only ports 80 and 443 are exposed to the internet; port 8080 stays local.

## 1. Auth0 configuration

- **Tenant:** `dev-xxjl6y6dsugnp3lf.us.auth0.com`
- **API — Enterprise Risk Lab API:** identifier (audience) `https://enterprise-risk-api`,
  signing algorithm RS256, permission `read:risk` (*Read the protected risk status*).
- **SPA — Enterprise Risk Lab Frontend:** allowed callback URLs, logout URLs and web origins
  set to `http://localhost:5173` and `https://danielenterpriseapplication.duckdns.org`.
- **User-Delegated Access:** the SPA was granted access to the API with the `read:risk`
  permission (application → APIs tab). A green check under *Client Access* is **not**
  enough — that is for machine-to-machine clients. Without user-delegated access Auth0
  returns `Client "..." is not authorized to access resource server "https://enterprise-risk-api"`.
- **Test user:** created in Authentication → Database → Users. The Auth0 dashboard account
  and the application users are different identity stores.

![Enterprise Risk Lab Frontend — SPA in Auth0](https://raw.githubusercontent.com/A1Daniel1/securityLab-frontend/main/image.png)

![Enterprise Risk Lab API — audience https://enterprise-risk-api](https://raw.githubusercontent.com/A1Daniel1/securityLab-frontend/main/image-1.png)

## 2. Backend — Spring Boot Resource Server

Repository: [securityLab-backend](https://github.com/A1Daniel1/securityLab-backend)

- Maven project, Java 21, dependencies: `spring-boot-starter-web`,
  `spring-boot-starter-security`, `spring-boot-starter-oauth2-resource-server`.
- `SecurityConfig` (`src/main/java/edu/eci/co/EnterpriseRiskLab/SecurityConfig.java`):
  CSRF disabled, CORS restricted to `http://localhost:5173` and
  `https://danielenterpriseapplication.duckdns.org`, `/actuator/health` public,
  `/api/**` authenticated, any other request denied, JWT validation enabled.
- `application.properties`:

  ```properties
  spring.security.oauth2.resourceserver.jwt.issuer-uri=https://dev-xxjl6y6dsugnp3lf.us.auth0.com/
  spring.security.oauth2.resourceserver.jwt.audiences=https://enterprise-risk-api
  server.port=8080
  ```

- `RiskController` exposes `GET /api/risk-status`, returning the protection status, the
  JWT subject, and the applied control.

Run locally:

```bash
./mvnw spring-boot:run
```

Unauthenticated requests are rejected with **401 Unauthorized**, as expected from a
protected resource server:

![curl to /api/risk-status without a token returns 401](https://raw.githubusercontent.com/A1Daniel1/securityLab-backend/main/image.png)

## 3. Frontend — React SPA

Repository: [securityLab-frontend](https://github.com/A1Daniel1/securityLab-frontend)
(application inside the `frontend/` directory)

`.env.local` (never committed):

```bash
VITE_AUTH0_DOMAIN=dev-xxjl6y6dsugnp3lf.us.auth0.com
VITE_AUTH0_CLIENT_ID=<spa-client-id>
VITE_AUTH0_AUDIENCE=https://enterprise-risk-api
VITE_API_URL=http://localhost:8080   # https://danielenterpriseapplication.duckdns.org for the production build
```

Run locally:

```bash
npm install
npm run dev
```

The app wraps the tree in `Auth0Provider` (with `audience`), redirects to Auth0 Universal
Login, obtains an access token with `getAccessTokenSilently()`, and calls the protected
endpoint with the `Authorization: Bearer <token>` header.

Login flow evidence:

![App served at localhost:5173 with Log in button](https://raw.githubusercontent.com/A1Daniel1/securityLab-frontend/main/image-2.png)

![Auth0 Universal Login for the SPA](https://raw.githubusercontent.com/A1Daniel1/securityLab-frontend/main/image-3.png)

![Authenticated session in the SPA](https://raw.githubusercontent.com/A1Daniel1/securityLab-frontend/main/image-4.png)

## 4. Deployment — AWS EC2 with Amazon Linux 2023

Equivalent of the workshop's Ubuntu steps, translated to the Amazon Linux 2023 AMI
(`dnf` packages, `httpd` service, `conf.d` virtual hosts, Amazon Corretto JDK).
The SSH default user is `ec2-user`.

### 4.1 Connect and install packages

```bash
# From the local machine (key permissions already fixed with icacls)
ssh -i aws/enterpriseDaniel.pem ec2-user@<EC2-PUBLIC-IP>

sudo dnf update -y
sudo dnf install -y httpd mod_ssl java-21-amazon-corretto-headless certbot python3-certbot-apache
```

### 4.2 Firewall

AWS replaces `ufw`: configure the instance **Security Group** with inbound rules for
TCP 80 and 443 only (plus SSH 22 restricted to your IP). Do **not** open port 8080.

Point the Duck DNS subdomain `danielenterpriseapplication.duckdns.org` to the instance
public IP — ideally an **Elastic IP** so it survives stop/start cycles.

### 4.3 Deploy the backend

```bash
# From the local machine
scp -i aws/enterpriseDaniel.pem securityLab-backend/target/*.jar ec2-user@<EC2-PUBLIC-IP>:/tmp/app.jar

# On the instance
sudo mkdir -p /opt/enterprise-risk-api
sudo useradd --system --no-create-home riskapi
sudo cp /tmp/app.jar /opt/enterprise-risk-api/app.jar
sudo chown -R riskapi:riskapi /opt/enterprise-risk-api
```

`/etc/systemd/system/enterprise-risk-api.service` (systemd works the same as on Ubuntu):

```ini
[Unit]
Description=Enterprise Risk API
After=network.target

[Service]
User=riskapi
WorkingDirectory=/opt/enterprise-risk-api
Environment="SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_ISSUER_URI=https://dev-xxjl6y6dsugnp3lf.us.auth0.com/"
Environment="SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_AUDIENCES=https://enterprise-risk-api"
ExecStart=/usr/bin/java -jar /opt/enterprise-risk-api/app.jar
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now enterprise-risk-api
curl -i http://127.0.0.1:8080/api/risk-status   # expect 401
```

### 4.4 Deploy the frontend

```bash
# From the local machine, after npm run build with VITE_API_URL=https://danielenterpriseapplication.duckdns.org
scp -i aws/enterpriseDaniel.pem -r securityLab-frontend/frontend/dist ec2-user@<EC2-PUBLIC-IP>:/tmp/dist

# On the instance
sudo mkdir -p /var/www/enterprise-risk-frontend
sudo cp -r /tmp/dist/* /var/www/enterprise-risk-frontend/
```

### 4.5 Apache (httpd) reverse proxy + SPA fallback

On Amazon Linux there is no `a2enmod`/`a2ensite`: `mod_ssl` comes from the `mod_ssl`
package, proxy/rewrite/headers modules are already loaded under
`/etc/httpd/conf.modules.d/`, and virtual hosts are dropped into `/etc/httpd/conf.d/`.

Create `/etc/httpd/conf.d/enterprise-risk.conf`:

```apache
<VirtualHost *:80>
    ServerName danielenterpriseapplication.duckdns.org
    DocumentRoot /var/www/enterprise-risk-frontend

    <Directory /var/www/enterprise-risk-frontend>
        Options -Indexes
        AllowOverride None
        Require all granted
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule ^ /index.html [L]
    </Directory>

    ProxyPreserveHost On
    ProxyPass /api/ http://127.0.0.1:8080/api/
    ProxyPassReverse /api/ http://127.0.0.1:8080/api/
</VirtualHost>
```

```bash
sudo httpd -t                      # equivalent of apachectl configtest
sudo systemctl enable --now httpd  # service is httpd, not apache2
# Only if SELinux is switched to Enforcing (proxy would fail with 502 otherwise):
# sudo setsebool -P httpd_can_network_connect 1
```

### 4.6 TLS certificate with Let's Encrypt

```bash
sudo certbot --apache -d danielenterpriseapplication.duckdns.org
sudo certbot renew --dry-run
```

Certbot creates the HTTPS virtual host, redirects port 80 → 443, and installs a renewal
timer.

## 5. Verification checklist

- [x] Unauthenticated requests to `/api/risk-status` return 401 (evidence in section 2).
- [x] A user can log in through Auth0 from the frontend (evidence in section 3).
- [x] The backend validates the JWT issuer and audience (`application.properties`).
- [x] The application is available at https://danielenterpriseapplication.duckdns.org.
- [x] The browser shows a valid Let's Encrypt certificate for the domain (evidence below).
- [x] Port 8080 is not reachable from the public internet (Security Group: inbound 80/443 only).
- [ ] `sudo certbot renew --dry-run` succeeds.

### Deployment evidence

Successful authenticated call to `GET /api/risk-status` — the API validated the Bearer
token and returned the JWT subject and the protection status to the signed-in user:

![Authenticated API call returning the protected JSON](https://raw.githubusercontent.com/A1Daniel1/securityLab-frontend/main/image-5.png)

Valid Let's Encrypt certificate issued for `danielenterpriseapplication.duckdns.org`:

![Valid Let's Encrypt certificate for the domain](https://raw.githubusercontent.com/A1Daniel1/securityLab-frontend/main/image-6.png)

## 6. Risk-control reflection

| Control | Risk reduced | Evidence |
|---|---|---|
| Auth0 authentication | Impersonation and weak credential handling | Auth0 login flow and user identity (section 3) |
| JWT validation at the API | Unauthorized API access | 401 without a valid token (section 2) |
| Audience validation | Token reuse against another API | API accepts only `https://enterprise-risk-api` |
| HTTPS with a valid certificate | Credential or token exposure in transit | Browser certificate details (section 5) |
| Apache reverse proxy | Direct exposure of the application port | Only ports 80/443 public; 8080 loopback-only |
| Central service logs | Delayed incident investigation | `journalctl -u enterprise-risk-api` |

## Security notes

- `.env.local`, the EC2 key pair (`enterpriseDaniel.pem`), Auth0 secrets and Duck DNS
  tokens are never committed.
- The SPA uses only the Auth0 client ID — an SPA must never hold a client secret.
- CORS is restricted to the exact development and production origins.
- Access tokens are sensitive: they are not displayed, logged, or stored manually.
- The backend runs as the unprivileged `riskapi` system user.
