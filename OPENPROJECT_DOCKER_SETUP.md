# OpenProject Docker Setup Guide

Runs OpenProject (stable `17-slim-bim`) with PostgreSQL 15 using Docker Compose.
On first boot it auto-runs migrations and creates an admin account.

---

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop) installed and running
- No extra tools required

---

## Quick Start

```bash
# 1. Copy or clone this folder to the target machine, then:
cd openproject_docker

# 2. Start everything
docker compose up -d

# 3. Watch startup (takes 60-120 s on first run  migrations + admin seed)
docker compose logs -f openproject-app
```

Wait until you see:
```
[1] * Listening on http://0.0.0.0:8080
Admin user created
```

Open **http://localhost:8080** and log in:

| Field    | Value      |
|----------|------------|
| Login    | `admin`    |
| Password | `admin123` |

> **Change the password immediately** after first login via the top-right menu ? *My account*.

---

## How it works

The `command:` in `docker-compose.yml` does three things in sequence before handing off to the web server:

1. **Runs all pending DB migrations** � `bundle exec rails db:migrate`
2. **Seeds the admin user** � only if it does not already exist (`find_or_initialize_by`)
3. **Starts the web server** � `./docker/prod/entrypoint-slim.sh ./docker/prod/web`

Because of step 2, **no manual admin setup is needed on any machine**.

---

## docker-compose.yml reference

| Key setting | Value | Purpose |
|---|---|---|
| `image` (db) | `postgres:15` | PostgreSQL database |
| `image` (app) | `openproject/openproject:17-slim-bim` | Stable OpenProject release |
| `condition: service_healthy` | � | App waits for DB to be ready before starting |
| `OPENPROJECT_HTTPS` | `false` | Disables HTTPS mode |
| `OPENPROJECT_HSTS` | `false` | Disables HSTS headers |
| `OPENPROJECT_HOST__NAME` | `localhost:8080` | Suppresses hostname mismatch warning |
| `SECRET_KEY_BASE` | (see file) | Rails secret � change for production |

---

## Environment variables

| Variable | Default | Purpose |
|---|---|---|
| `POSTGRES_DB` / `POSTGRES_USER` / `POSTGRES_PASSWORD` | `openproject` | DB credentials |
| `DATABASE_URL` | (matches above) | Connection string for the app |
| `SECRET_KEY_BASE` | `local-openproject-secret-key-change-me` | Change for any non-local deployment |
| `OPENPROJECT_HTTPS` | `false` | HTTP-only mode |
| `OPENPROJECT_HSTS` | `false` | No HSTS headers |
| `OPENPROJECT_HOST__NAME` | `localhost:8080` | Sets expected hostname |

---

## Common commands

```bash
# Start
docker compose up -d

# Stop (keeps data)
docker compose stop

# Start again after stop
docker compose start

# View live logs
docker compose logs -f openproject-app

# Check container status
docker compose ps

# Full reset � DELETES ALL DATA
docker compose down -v
docker compose up -d
```

---

## Troubleshooting

### Login fails � "invalid credentials"

The admin password was changed or the seed did not run. Reset it:

```bash
docker exec openproject-app bash -c "cat > /tmp/reset_pass.rb << 'SCRIPT'
u = User.find_by(login: 'admin')
u.passwords.delete_all
pw = UserPassword::Bcrypt.new(user: u, plain_password: 'admin123')
pw.save!
puts 'Password reset for: ' + u.login
SCRIPT"

docker exec openproject-app bash -c "cd /app && RAILS_ENV=production bundle exec rails runner /tmp/reset_pass.rb"
```

### 500 error after login

Check logs:
```bash
docker compose logs --tail=30 openproject-app
```

If you see `single-table inheritance mechanism failed`, the `user_passwords.type` column has an invalid class name.
Run the password reset commands above.

### Port 8080 already in use

Change the host port in `docker-compose.yml`:
```yaml
ports:
  - "9000:8080"   # access at http://localhost:9000
```

Also update `OPENPROJECT_HOST__NAME: "localhost:9000"`.

### Browser redirects to HTTPS

Clear HSTS cache:
- **Chrome / Edge**: Go to `chrome://net-internals/#hsts` ? delete `localhost`
- **Firefox**: Settings ? Privacy ? Manage Data ? remove `localhost`
- Or use an incognito window

### "PG::ConnectionBad: role does not exist"

Old volume from a different DB user. Full reset:
```bash
docker compose down -v
docker compose up -d
```

### Slow first start

Normal � migrations run only once. Subsequent starts are fast (< 10 s).
Ensure Docker Desktop has at least **4 GB RAM** (Settings ? Resources).

---

## Upgrading OpenProject

1. Change the image tag in `docker-compose.yml` (e.g. `17-slim-bim` ? `18-slim-bim`)
2. Restart � migrations run automatically:
   ```bash
   docker compose up -d
   ```
