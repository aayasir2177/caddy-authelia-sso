# Caddy Authelia SSO Architecture

Centralized Single Sign-On (SSO) forward-authentication architecture securing local containerized services (`*.home.server`) using Caddy reverse proxy and Authelia.

---

## Architecture Overview

When a client requests access to a service (e.g., `dockhand.home.server`):

1. **Caddy** receives the incoming HTTP request.
2. **Caddy** queries Authelia's `/api/verify` endpoint using `forward_auth`.
3. **Authelia** validates the session cookie:
   - **Authenticated**: Authelia returns `200 OK`, and Caddy forwards traffic to the backend service.
   - **Unauthenticated**: Authelia returns `401 Unauthorized`, and Caddy redirects the client to `auth.home.server`.

---

## Screenshots

<img width="1919" height="881" alt="authelia" src="https://github.com/user-attachments/assets/6fc70631-7dc0-4b89-9c3e-5300abc980da" />
*Centralized Authelia Authentication Portal at `auth.home.server`.*

<img width="1866" height="849" alt="authlia graph" src="https://github.com/user-attachments/assets/98049a00-98f6-4172-92af-efe1e419e276" />
*Authelia Graph.*

<img width="946" height="958" alt="caddy" src="https://github.com/user-attachments/assets/7867bcdd-d4ed-4e07-9586-b299b2a043f3" />
*Caddy Config file.*

---

## Configuration Files

### 1. Caddyfile

```json
(authelia_auth) {
    forward_auth authelia:9091 {
        uri /api/verify
        copy_headers Remote-User Remote-Groups Remote-Name Remote-Email
    }
}

# Authentication Portal
auth.home.server {
    tls internal
    reverse_proxy authelia:9091
}

# Management Services (SSO Protected)
dockhand.home.server {
    tls internal
    import authelia_auth
    reverse_proxy dockhand:3000
}

beszel.home.server {
    tls internal
    import authelia_auth
    reverse_proxy beszel:8090
}

# User Services (SSO Protected)
qbt.home.server {
    tls internal
    import authelia_auth
    reverse_proxy qbittorrent:8080
}

# Standalone Services (Native Auth)
vault.home.server {
    tls internal
    reverse_proxy vaultwarden:80
}
```

### 2. Authelia Configuration

```yaml
server:
  host: 0.0.0.0
  port: 9091

session:
  name: authelia_session
  domain: home.server
  same_site: lax
  expiration: 1h
  inactivity: 5m
  remember_me_duration: 1M

access_control:
  default_policy: deny
  rules:
    - domain: "auth.home.server"
      policy: bypass
    - domain: "*.home.server"
      policy: one_factor

authentication_backend:
  file:
    path: /config/users_database.yml
```

### 3. Users Database 

```yaml
users:
  aayasir217:
    disabled: false
    displayname: "Yasir"
    password: '$argon----d...19$------,t=3,p=4$...' #i changed the hashed pass
    email: "aayasir217@gmail.com"
    groups:
      - "admins"
      - "devs"
```


