# Credentials Setup Guide

All credentials used by this OpenProject Docker setup, what each one does, and how to
change it before deploying to a new environment.

---

## Summary table

| Credential | Where it lives | User-facing? | Must change for prod? |
|---|---|---|---|
| `POSTGRES_USER` | compose file | No (internal) | Recommended |
| `POSTGRES_PASSWORD` | compose file | No (internal) | **Yes** |
| `POSTGRES_DB` | compose file | No (internal) | Optional |
| `DATABASE_URL` | compose file | No (internal) | **Yes** (must match above) |
| `SECRET_KEY_BASE` | compose file | No (internal) | **Yes** |
| Admin password (`admin123`) | compose `command:` | **Yes** (login) | **Yes** |
| `OPENPROJECT_HOST__NAME` | compose file | No | **Yes** (per machine) |

---

## 1. Database credentials

### What they are
`POSTGRES_USER`, `POSTGRES_PASSWORD`, and `POSTGRES_DB` control who owns the PostgreSQL
database. `DATABASE_URL` is the connection string the app uses — it must always match those
three values.

### Where to change
In `docker-compose.yml`, edit **4 places** consistently:
```yaml
# openproject-db service
POSTGRES_DB: <your_db>
POSTGRES_USER: <your_user>
POSTGRES_PASSWORD: <your_password>

# openproject-app service
DATABASE_URL: postgresql://<your_user>:<your_password>@openproject-db:5432/<your_db>
```

> After changing DB credentials, always do a full volume reset or the old user/DB will still be on disk:
> ```bash
> docker compose down -v
> docker compose up -d
> ```

### Generating a random password

**PowerShell:**
```powershell
-join ((1..24) | ForEach-Object { [char](Get-Random -Minimum 65 -Maximum 91) + [char](Get-Random -Minimum 97 -Maximum 123) + (Get-Random -Maximum 10) })
```

**CMD** (calls PowerShell internally):
```cmd
powershell -command "-join ((1..24) | ForEach-Object { [char](Get-Random -Minimum 65 -Maximum 91) + (Get-Random -Maximum 10) })"
```

**Linux / macOS (bash):**
```bash
openssl rand -base64 24
```

---

## 2. SECRET_KEY_BASE

### What it is
The Rails application secret. Used internally to sign session cookies, CSRF tokens, and
other framework-level data. You never type this anywhere — it just must exist and stay stable
for a given environment.

### Rules
- Must be at least 30 characters; 64+ hex characters recommended.
- Never reuse the same value across different environments.
- If you change it on a running instance, all active sessions are invalidated (everyone is
  logged out). On a fresh setup that is fine.

### Where to change
In `docker-compose.yml`:
```yaml
SECRET_KEY_BASE: <your_generated_secret>
```

### Generating a secure value

**PowerShell:**
```powershell
-join ((1..64) | ForEach-Object { '{0:x}' -f (Get-Random -Maximum 16) })
```

**CMD** (calls PowerShell internally):
```cmd
powershell -command "-join ((1..64) | ForEach-Object { '{0:x}' -f (Get-Random -Maximum 16) })"
```

**Linux / macOS (bash):**
```bash
openssl rand -hex 64
```

Copy the output and put it in the compose file before the first `docker compose up -d`.

---

## 3. Admin login password

### What it is
The password for the `admin` OpenProject user created automatically on first boot.
This is the password you type on the login page.

### Where to change
In `docker-compose.yml`, inside the `command:` block — find `plain_password: "admin123"` and
replace `admin123` with your chosen password:

```yaml
command:
  - sh
  - -c
  - |
    bundle exec rails db:migrate RAILS_ENV=production &&
    bundle exec rails runner -e production 'u = User.find_or_initialize_by(login: "admin"); unless u.persisted?; u.assign_attributes(...); u.save!(validate: false); pw = UserPassword::Bcrypt.new(user: u, plain_password: "YOUR_PASSWORD_HERE"); pw.save!; puts "Admin user created"; end' &&
    ./docker/prod/entrypoint-slim.sh ./docker/prod/web
```

> This only runs on first boot (when the admin user does not exist yet). To reset the
> password on an already-running instance, see below.

### Resetting the admin password on a running instance

**PowerShell / CMD / Linux / macOS — same command for all:**
```bash
docker exec openproject-app bash -c "cat > /tmp/reset_pass.rb << 'SCRIPT'
u = User.find_by(login: 'admin')
u.passwords.delete_all
pw = UserPassword::Bcrypt.new(user: u, plain_password: 'NEW_PASSWORD_HERE')
pw.save!
puts 'Password reset for: ' + u.login
SCRIPT"

docker exec openproject-app bash -c "cd /app && RAILS_ENV=production bundle exec rails runner /tmp/reset_pass.rb"
```

Replace `NEW_PASSWORD_HERE` with the desired password.

---

## 4. OPENPROJECT_HOST__NAME

### What it is
Tells OpenProject what hostname and port to expect in incoming requests. If it does not
match what the browser sends, a warning banner appears on every page.

### Where to change
In `docker-compose.yml`:
```yaml
OPENPROJECT_HOST__NAME: "localhost:8080"
```

Replace `localhost:8080` with the actual hostname and port of the machine:

| Machine type | Example value |
|---|---|
| Local (default) | `localhost:8080` |
| Local on different port | `localhost:9000` |
| Server with hostname | `openproject.mycompany.com` |
| Server with IP | `192.168.1.50:8080` |

No restart of volumes is needed — just `docker compose up -d` to apply.

---

## Quick checklist for a new environment

1. **Generate a new `SECRET_KEY_BASE`** (see section 2) and replace the placeholder.
2. **Set `OPENPROJECT_HOST__NAME`** to match the machine's hostname/IP and port.
3. **Optionally change DB credentials** (POSTGRES_USER, POSTGRES_PASSWORD, DATABASE_URL).
4. **Optionally change the default admin password** in the `command:` block.
5. Run:
   ```bash
   docker compose up -d
   docker compose logs -f openproject-app
   ```
6. Log in at `http://<hostname>:8080` with `admin` / the password from step 4.
7. Change the admin password via *My account* after first login.
