# OpenProject Docker Setup Guide

This guide walks you through setting up OpenProject with PostgreSQL using Docker and Docker Compose. By the end, you'll have a fully functional project management platform running locally.

## Prerequisites

- **Docker Desktop** installed on your machine ([Download here](https://www.docker.com/products/docker-desktop))
- **Docker Compose** (included with Docker Desktop)
- A text editor (VS Code, Notepad++, etc.)
- Internet connection to download images (~800MB for first run)

## Docker Concepts Explained

### What is Docker?

Docker is a containerization platform that packages applications and their dependencies into isolated, portable units called **containers**. Think of it as a lightweight virtual machine that runs an application exactly the same way everywhere.

**Key Benefits:**
- **Consistency**: Works the same on your laptop, a coworker's machine, or a production server
- **Isolation**: Each container has its own environment, so apps don't interfere with each other
- **Simplicity**: No complex installation procedures; just run a command

📖 [Learn more: Docker Official Guide](https://docs.docker.com/get-started/)

### What is Docker Compose?

Docker Compose lets you define and run **multiple containers** together using a single YAML file (`docker-compose.yml`). Instead of running separate `docker run` commands for each service, you define everything in one place.

In this guide, we're running two containers:
1. **OpenProject** (the web application)
2. **PostgreSQL** (the database)

Docker Compose connects them on the same network so they can talk to each other.

📖 [Learn more: Docker Compose Documentation](https://docs.docker.com/compose/)

### Docker Images vs Containers

- **Image**: A blueprint or template (like a recipe)
- **Container**: A running instance of an image (like a cooked dish)

When you run `docker run openproject/openproject:dev-slim-bim`, Docker takes the image and creates a container from it.

### Networks

Containers on the same Docker network can communicate using service names as hostnames. In our setup, the OpenProject container can reach the database at `openproject-db:5432` automatically.

📖 [Learn more: Docker Networking](https://docs.docker.com/network/)

### Volumes

Volumes are persistent storage locations that survive container restarts. Without volumes, your database would be deleted when the container stops. We use volumes to store:
- Database data (PostgreSQL)
- Application assets (OpenProject)

📖 [Learn more: Docker Volumes](https://docs.docker.com/storage/volumes/)

## Step-by-Step Setup

### Step 1: Create a Project Directory

Create a folder for your OpenProject setup:

```bash
mkdir openproject-docker
cd openproject-docker
```

### Step 2: Create the docker-compose.yml File

Create a new file named `docker-compose.yml` in your project folder with the following content:

```yaml
services:
  openproject-db:
    image: postgres:15
    container_name: openproject-db
    environment:
      POSTGRES_USER: ffribeiro
      POSTGRES_PASSWORD: 1234
      POSTGRES_DB: openproject
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - openproject-network

  openproject-app:
    image: openproject/openproject:dev-slim-bim
    container_name: openproject-app
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgresql://ffribeiro:1234@openproject-db:5432/openproject
      OPENPROJECT_HTTPS: "false"
      OPENPROJECT_HSTS: "false"
      OPENPROJECT_FORCE_HTTPS: "false"
      RAILS_ENV: production
      PUMA_BIND: "tcp://0.0.0.0:8080"
    depends_on:
      - openproject-db
    networks:
      - openproject-network
    volumes:
      - openproject_assets:/var/openproject/assets
    command: sh -c "bundle exec rails db:migrate RAILS_ENV=production && ./docker/prod/entrypoint-slim.sh ./docker/prod/web"

volumes:
  postgres_data:
  openproject_assets:

networks:
  openproject-network:
    driver: bridge
```

**Key changes from earlier:**
- Removed obsolete `version: '3.8'` line
- Added `command:` to automatically run database migrations on startup

**What each section does:**

#### `services:`
Defines the containers to run. We have two: a database and the application.

#### `openproject-db:` (PostgreSQL Service)
- **image**: Uses PostgreSQL version 15
- **environment**: Sets up database credentials (username, password, database name)
- **ports**: Maps port 5432 inside the container to port 5432 on your machine
- **volumes**: Persists database data so it survives restarts
- **networks**: Connects to the custom network

#### `openproject-app:` (OpenProject Service)
- **image**: The OpenProject application image (dev-slim-bim variant)
- **ports**: Maps port 8080 inside the container to 8080 on your machine (access via http://localhost:8080)
- **environment**: Configuration variables:
  - `DATABASE_URL`: Connection string to PostgreSQL
  - `OPENPROJECT_HTTPS: "false"`: Disables HTTPS to use HTTP
  - `OPENPROJECT_HSTS: "false"`: Disables security headers that force HTTPS
  - `RAILS_ENV: production`: Runs in production mode
- **depends_on**: Waits for the database to start first
- **networks**: Connects to the same network as the database
- **volumes**: Persists application assets
- **command**: Runs database migrations automatically before starting the app

#### `volumes:` and `networks:`
Define the shared storage and network for all containers.

📖 [Learn more: docker-compose.yml Reference](https://docs.docker.com/compose/compose-file/)

### Step 3: Start the Containers

Open a terminal/command prompt in your `openproject-docker` folder and run:

```bash
docker compose up -d
```

**What this does:**
- `-d`: Runs containers in the background (detached mode)
- Downloads the images if not already present
- Creates and starts both containers
- Automatically runs database migrations via the `command:` directive

**Expected output:**
```
Creating network openproject-network
Creating volume postgres_data
Creating volume openproject_assets
Creating openproject-db
Creating openproject-app
```

### Step 4: Monitor the Initialization Process

The first run takes **1-2 minutes** because it's downloading images and running 92 database migrations. Monitor progress with:

```bash
docker compose logs -f openproject-app
```

**What this does:**
- `-f`: "Follow" mode - shows live logs as they happen
- Press `Ctrl + C` to stop watching logs

**What to look for:**

Initial output (migrations running):
```
I, [2026-04-15T01:27:30.427889 #7]  INFO -- : Migrating to AggregatedMigrations (1000016)
== 1000016 AggregatedMigrations: migrating ====================================
-- execute("CREATE EXTENSION IF NOT EXISTS btree_gist WITH SCHEMA pg_catalog;")
   -> 0.0535s
```

Success message (when ready):
```
[1] * Listening on http://0.0.0.0:8080
[1] Use Ctrl-C to stop
```

**If migrations get stuck:** This is normal. The first migration (`MigrateClassicMeetings`) can take 15+ seconds. Wait and watch—don't interrupt.

**Alternative - Check status without following logs:**
```bash
docker compose logs openproject-app
```

### Step 5: Verify Containers Are Running

Once migrations complete, verify both containers are healthy:

```bash
docker compose ps
```

You should see:
```
NAME               STATUS
openproject-db     Up 2 minutes
openproject-app    Up 1 minute
```

### Step 6: Access OpenProject

Open your browser and go to:

```
http://localhost:8080
```

You should see the OpenProject login page.

### Step 7: Clear Browser HSTS Cache (If Redirected to HTTPS)

If your browser redirects to HTTPS, it's cached an old security policy. Clear the HSTS cache:

**Chrome/Chromium:**
1. Go to `chrome://net-internals/#hsts`
2. Under "Delete domain security policies", type `localhost`
3. Click **Delete**
4. Refresh the page

**Firefox:**
1. Go to **Settings** → **Privacy & Security**
2. Scroll to **Cookies and Site Data**
3. Click **Manage Data** → Search for `localhost` → **Remove All**

**Edge:**
1. Go to `edge://net-internals/#hsts`
2. Same steps as Chrome

**Alternative:** Open in an incognito/private window (doesn't have cached HSTS).

## Common Commands

### View Running Containers

```bash
docker compose ps
```

### View Logs (Live)

Watch logs in real-time as they happen:
```bash
docker compose logs -f openproject-app
```

Press `Ctrl + C` to stop watching.

### View Logs (Static)

Show recent logs without following:
```bash
docker compose logs openproject-app
docker compose logs openproject-db
```

Show all logs:
```bash
docker compose logs
```

### Stop Containers

```bash
docker compose stop
```

Containers stop but data persists. Restart with `docker compose start`.

### Start Containers (After Stopping)

```bash
docker compose start
```

### Restart Containers

```bash
docker compose restart
```

Useful if you change environment variables.

### Stop and Remove Everything (Keep Data)

```bash
docker compose down
```

Removes containers and networks but keeps volumes. Perfect for cleaning up while preserving data.

### Stop and Remove Everything (Including Data - Fresh Start)

```bash
docker compose down -v
```

⚠️ **Warning**: Using `-v` deletes all data (database, assets). Use this only if you want a completely fresh start.

## Environment Variables Explained

In the `docker-compose.yml`, the `environment:` section sets configuration variables:

| Variable | Value | Purpose |
|----------|-------|---------|
| `POSTGRES_USER` | `ffribeiro` | Database username |
| `POSTGRES_PASSWORD` | `1234` | Database password |
| `POSTGRES_DB` | `openproject` | Initial database name |
| `DATABASE_URL` | Connection string | How OpenProject connects to PostgreSQL |
| `OPENPROJECT_HTTPS` | `false` | Disables HTTPS mode (allows HTTP) |
| `OPENPROJECT_HSTS` | `false` | Disables HSTS security headers |
| `OPENPROJECT_FORCE_HTTPS` | `false` | Prevents forced HTTPS redirects |
| `RAILS_ENV` | `production` | Runs in production mode (optimized) |

**To change credentials:**
1. Edit `docker-compose.yml`
2. Update `POSTGRES_USER`, `POSTGRES_PASSWORD`, and the `DATABASE_URL` (must match)
3. Run `docker compose down -v` to delete old data
4. Run `docker compose up -d` to start fresh with new credentials

## Troubleshooting

### Containers Won't Start / Initialization Hangs

Check logs to see what's happening:
```bash
docker compose logs -f openproject-app
```

Common issues:
- **First migration takes time**: The `MigrateClassicMeetings` migration can take 10-20 seconds. Be patient—don't interrupt.
- **Port 8080 already in use**: Change `"8080:8080"` to `"9000:8080"` in the compose file, then access at `http://localhost:9000`
- **Port 5432 already in use**: Change `"5432:5432"` to `"5433:5432"` and update `DATABASE_URL` to use port 5433

### Migrations Error / PendingMigrationError

This means the migrations didn't run. Fix it:

```bash
docker compose down -v
docker compose up -d
```

The `-v` flag deletes old data and starts fresh.

### Database Connection Error

Make sure containers are running:
```bash
docker compose ps
```

If `openproject-db` is not running, check its logs:
```bash
docker compose logs openproject-db
```

Common cause: PostgreSQL port 5432 is already in use. Change it in the compose file.

### HTTPS Redirect Loop

Your browser has cached an HSTS (HTTP Strict Transport Security) policy. Clear it (see Step 7 above) or use incognito mode.

### Slow Performance

Docker Desktop on Windows/Mac uses virtualization. For better performance:
- Ensure Docker Desktop has sufficient CPU/memory (Settings → Resources)
- Use WSL 2 backend on Windows instead of Hyper-V
- Close unnecessary applications

### Container Crashes / "Exit code 137"

Usually means out of memory. Increase Docker's memory allocation:
1. Open Docker Desktop settings
2. Go to **Resources**
3. Increase **Memory** slider (recommend 4GB+)
4. Click **Apply & Restart**

## Next Steps

1. **Log in**: Default credentials are created during first run. Check OpenProject documentation for initial setup.
2. **Create Projects**: Start creating work projects and managing tasks.
3. **Configure Settings**: Go to Administration to customize OpenProject.
4. **Backup Data**: Periodically back up your `postgres_data` volume.

📖 [OpenProject Documentation](https://www.openproject.org/docs/)

## Security Notes

⚠️ **This setup is for local development only!** For production:
- Change default credentials to strong passwords
- Use environment variable files (`.env`) instead of hardcoding them
- Enable HTTPS with proper SSL certificates
- Use a reverse proxy (nginx) in front of OpenProject
- Set resource limits on containers
- Enable automatic backups
- Use secrets management for sensitive data

📖 [Docker Security Best Practices](https://docs.docker.com/engine/security/)

## Useful Resources

- [Docker Official Documentation](https://docs.docker.com/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [PostgreSQL Docker Image](https://hub.docker.com/_/postgres)
- [OpenProject Docker Documentation](https://www.openproject.org/docs/installation-and-operations/installation/)
- [Docker Networking Guide](https://docs.docker.com/network/)
- [Docker Volumes Guide](https://docs.docker.com/storage/volumes/)
- [Docker Troubleshooting](https://docs.docker.com/config/containers/logging/)
