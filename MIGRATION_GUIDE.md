# Docker Container to Docker Compose Migration Guide
## Migrating Lucee ColdFusion + PostgreSQL to Docker Compose

**Author**: Kwame (Akimtech, LLC)  
**Date**: December 31, 2025  
**Purpose**: Step-by-step guide to migrate legacy Docker containers to Docker Compose

---

## Overview

This guide migrates standalone Docker containers (created with `docker run`) to a managed Docker Compose setup with:
- Configuration persistence
- Easy multi-container orchestration
- Environment variable management
- Reproducible deployments

---

## Prerequisites

- Docker and Docker Compose installed
- Existing Lucee and PostgreSQL containers running
- Basic terminal/command line knowledge

---

## Step 1: Document Your Current Setup

### 1.1 List Running Containers

```bash
# View all running containers
docker ps

# View all containers (including stopped)
docker ps -a
```

### 1.2 Document Current Container Commands

Find your original `docker run` commands or inspect existing containers:

```bash
# Inspect Lucee container
docker inspect <lucee-container-name>

# Inspect PostgreSQL container
docker inspect <postgres-container-name>
```

**Example Legacy Commands:**

```bash
# Legacy Lucee Container
docker run -d \
  -p 8889:8888 \
  -e LUCEE_ADMIN_PASSWORD=kwame123 \
  -v /Users/username/Documents/appdev/www:/var/www \
  --name mylucee \
  lucee/lucee

# Legacy PostgreSQL Container
docker run -d \
  -e POSTGRES_PASSWORD=kwame123 \
  -p 15432:5432 \
  -e POSTGRES_USER=appdev \
  -v /Users/username/Documents/appdev/data:/var/lib/postgresql/data \
  --name mypostgres \
  postgres
```

---

## Step 2: Create Project Directory Structure

```bash
# Navigate to your project directory
cd /Users/username/Documents/appdev

# Create configuration directories
mkdir -p lucee-config/server
mkdir -p lucee-config/web
mkdir -p www
mkdir -p data

# Verify structure
ls -la
```

**Expected Structure:**

```
/Users/username/Documents/appdev/
├── docker-compose.yml (to be created)
├── .env (to be created)
├── lucee-config/
│   ├── server/ (Lucee server configs)
│   └── web/ (Lucee web context configs)
├── www/ (Your ColdFusion applications)
└── data/ (PostgreSQL data - may already exist)
```

---

## Step 3: Backup Existing Lucee Configurations

### 3.1 Create Backup Directory

```bash
mkdir -p lucee-backup
```

### 3.2 Copy Configs from Running Container

```bash
# Replace <lucee-container-name> with your actual container name
LUCEE_CONTAINER="mylucee"

# Backup server configs
docker cp $LUCEE_CONTAINER:/opt/lucee/server lucee-backup/

# Backup web context configs
docker cp $LUCEE_CONTAINER:/opt/lucee/web lucee-backup/
```

### 3.3 Copy to New Config Location

```bash
# Copy server configs
cp -r lucee-backup/server lucee-config/

# Copy web configs (may be empty - that's OK)
cp -r lucee-backup/web lucee-config/

# Fix permissions
chmod -R 755 lucee-config/
```

**Note:** If you get "No such file or directory" errors for `cacerts`, that's normal and safe to ignore. Lucee will regenerate it.

---

## Step 4: Backup PostgreSQL Database (Optional but Recommended)

```bash
# Replace <postgres-container-name> with your actual container name
POSTGRES_CONTAINER="mypostgres"

# Full database backup
docker exec $POSTGRES_CONTAINER pg_dumpall -U appdev > full-backup-$(date +%Y%m%d).sql

# Backup specific database
docker exec $POSTGRES_CONTAINER pg_dump -U appdev appdev > appdev-backup-$(date +%Y%m%d).sql
```

---

## Step 5: Create .env File

Create `.env` file in your project directory:

```bash
cd /Users/username/Documents/appdev
nano .env
```

**Template .env File:**

```bash
# .env
# ============================================
# Lucee Configuration
# ============================================
LUCEE_ADMIN_PASSWORD=your_password_here
LUCEE_PORT=8889
LUCEE_ADMIN_ENABLED=true
TIMEZONE=America/New_York

# ============================================
# Application Paths (UPDATE THESE!)
# ============================================
APP_PATH=/Users/username/Documents/appdev/www
LUCEE_SERVER_CONFIG=./lucee-config/server
LUCEE_WEB_CONFIG=./lucee-config/web

# ============================================
# PostgreSQL Configuration
# ============================================
POSTGRES_USER=appdev
POSTGRES_PASSWORD=your_password_here
POSTGRES_DB=appdev
POSTGRES_PORT=15432
POSTGRES_DATA_PATH=/Users/username/Documents/appdev/data

# ============================================
# Database Connection (for Lucee)
# ============================================
DB_HOST=postgres
DB_PORT=5432
DB_NAME=appdev
DB_USER=appdev
DB_PASSWORD=your_password_here
```

**Important:** Update the following:
- Replace `/Users/username/` with your actual username
- Replace `your_password_here` with your actual passwords
- Update port numbers if different
- Update database names if different

Save and exit: `Ctrl+X`, then `Y`, then `Enter`

---
## Step 6A: Create docker-compose.yml file (Volumn on User Drive)

services:
  mylucee:
    build:      # This will use the Dockerfile in the current directory
      context: .
      dockerfile: Dockerfile
    container_name: mylucee
    ports:
      - "${LUCEE_PORT}:8888"
    volumes:
      - ${APP_PATH}:/var/www
      # Mount local directories to preserve configs
      - ${LUCEE_SERVER_CONFIG}:/opt/lucee/server
      - ${LUCEE_WEB_CONFIG}:/opt/lucee/web
    environment:
      - LUCEE_ADMIN_PASSWORD=${LUCEE_ADMIN_PASSWORD}
      - LUCEE_ADMIN_ENABLED=${LUCEE_ADMIN_ENABLED}
      - TZ=${TIMEZONE}
      # Database connection variables available to Lucee
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - lucee-network

  postgres:
    image: postgres:latest
    container_name: lucee-postgres
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      # Your existing PostgreSQL data - SAFE!
      - ${POSTGRES_DATA_PATH}:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - lucee-network

networks:
  lucee-network:
    driver: bridge
    
## Step 6: Create docker-compose.yml File

```bash
nano docker-compose.yml
```

**Template docker-compose.yml:**

```yaml
version: '3.8'

services:
  mylucee:
    image: lucee/lucee:latest
    container_name: mylucee
    ports:
      - "${LUCEE_PORT}:8888"
    volumes:
      - ${APP_PATH}:/var/www
      - ${LUCEE_SERVER_CONFIG}:/opt/lucee/server
      - ${LUCEE_WEB_CONFIG}:/opt/lucee/web
    environment:
      - LUCEE_ADMIN_PASSWORD=${LUCEE_ADMIN_PASSWORD}
      - LUCEE_ADMIN_ENABLED=${LUCEE_ADMIN_ENABLED}
      - TZ=${TIMEZONE}
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
      - DB_NAME=${DB_NAME}
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - lucee-network

  postgres:
    image: postgres:latest
    container_name: lucee-postgres
    ports:
      - "${POSTGRES_PORT}:5432"
    volumes:
      - ${POSTGRES_DATA_PATH}:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - lucee-network

networks:
  lucee-network:
    driver: bridge
```

Save and exit: `Ctrl+X`, then `Y`, then `Enter`

---

## Step 7: Validate Configuration

```bash
# Verify .env file exists and has correct paths
cat .env

# Verify docker-compose.yml exists
cat docker-compose.yml

# Test docker-compose configuration (shows resolved values)
docker-compose config
```

---

## Step 8: Stop Legacy Containers

**CRITICAL:** Stop old containers to avoid port conflicts

```bash
# List running containers
docker ps

# Stop legacy containers (replace with your actual container names)
docker stop mylucee
docker stop mypostgres

# Optional: Remove legacy containers (data is safe in volumes)
docker rm mylucee
docker rm mypostgres
```

**Data Safety Check:**
- ✅ Application code is in `/Users/username/Documents/appdev/www`
- ✅ PostgreSQL data is in `/Users/username/Documents/appdev/data`
- ✅ Lucee configs are backed up to `lucee-config/`

Removing containers does NOT delete your data!

---

## Step 9: Start Services with Docker Compose

```bash
cd /Users/username/Documents/appdev

# Start services in detached mode
docker-compose up -d

# Watch logs (Ctrl+C to exit)
docker-compose logs -f

# Check status
docker-compose ps
```

**Expected Output:**

```
[+] Running 3/3
 ✔ Network appdev_lucee-network  Created
 ✔ Container lucee-postgres      Started (healthy)
 ✔ Container mylucee             Started
```

---

## Step 10: Verify Everything Works

### 10.1 Check Container Status

```bash
docker-compose ps
```

Should show:
```
NAME              STATUS         PORTS
lucee-postgres    Up (healthy)   0.0.0.0:15432->5432/tcp
mylucee           Up             0.0.0.0:8889->8888/tcp
```

### 10.2 Access Lucee Admin

Open browser: `http://localhost:8889/lucee/admin/web.cfm`

- Login with password from `.env` file
- Verify datasources under **Services → Datasource**
- Check installed extensions under **Extension → Applications**

### 10.3 Test PostgreSQL Connection

```bash
# From host machine
psql -h localhost -p 15432 -U appdev -d appdev

# From container
docker-compose exec postgres psql -U appdev -d appdev

# List databases
docker-compose exec postgres psql -U appdev -d appdev -c "\l"

# List tables
docker-compose exec postgres psql -U appdev -d appdev -c "\dt"
```

### 10.4 Test Your Application

Open browser: `http://localhost:8889/`

Navigate to your apps and verify they work.

---

## Step 11: Configure Datasource in Lucee (If Needed)

### Option A: Via Lucee Admin

1. Go to `http://localhost:8889/lucee/admin/web.cfm`
2. Navigate to **Services → Datasource**
3. Click **Add Datasource**
4. Configure:
   - **Name**: `appdev`
   - **Type**: PostgreSQL
   - **Host**: `postgres` (service name, not localhost!)
   - **Port**: `5432` (internal port, not 15432!)
   - **Database**: `appdev`
   - **Username**: `appdev`
   - **Password**: (from your .env)
5. Click **Create**

### Option B: Via Application.cfc (Recommended)

Create or update `Application.cfc` in your app root:

```cfml
<cfcomponent output="false">
    <cfset this.name = "MyApp">
    <cfset this.sessionManagement = true>
    
    <cfscript>
        // Get environment variables from docker-compose
        dbHost = server.system.environment.DB_HOST ?: "postgres";
        dbPort = server.system.environment.DB_PORT ?: "5432";
        dbName = server.system.environment.DB_NAME ?: "appdev";
        dbUser = server.system.environment.DB_USER ?: "appdev";
        dbPass = server.system.environment.DB_PASSWORD ?: "password";
        
        this.datasources = {
            appdev: {
                class: 'org.postgresql.Driver',
                connectionString: 'jdbc:postgresql://#dbHost#:#dbPort#/#dbName#',
                username: dbUser,
                password: dbPass
            }
        };
        
        this.datasource = "appdev";
    </cfscript>
</cfcomponent>
```

---

## Docker Compose Command Reference

### Starting and Stopping

```bash
# Start services (creates containers if needed)
docker-compose up -d

# Stop services (keeps containers)
docker-compose stop

# Stop and remove containers (keeps data)
docker-compose down

# Stop, remove containers AND volumes (⚠️ DELETES DATA!)
docker-compose down -v
```

### Viewing Logs

```bash
# View all logs
docker-compose logs

# View specific service logs
docker-compose logs mylucee
docker-compose logs postgres

# Follow logs in real-time
docker-compose logs -f

# Last 100 lines
docker-compose logs --tail=100
```

### Container Management

```bash
# View running containers
docker-compose ps

# Restart specific service
docker-compose restart mylucee

# Rebuild and restart (after code changes)
docker-compose up -d --build

# Execute command in container
docker-compose exec mylucee bash
docker-compose exec postgres psql -U appdev
```

### Maintenance

```bash
# Pull latest images
docker-compose pull

# Remove stopped containers
docker-compose rm

# View resource usage
docker-compose stats
```

---

## Troubleshooting

### Port Conflicts

**Error:** `bind: address already in use`

**Solution:**
```bash
# Find what's using the port
lsof -i :8889
lsof -i :15432

# Stop the conflicting container
docker stop <container-name>

# Or change ports in .env file
```

### Permission Errors

**Error:** `permission denied`

**Solution:**
```bash
# Fix permissions on config directories
chmod -R 755 lucee-config/
chmod -R 755 www/

# Fix ownership (if needed)
sudo chown -R $(whoami):staff lucee-config/
```

### Container Won't Start

**Solution:**
```bash
# View detailed logs
docker-compose logs mylucee

# Check if volumes exist
ls -la /path/to/volumes

# Restart with fresh containers
docker-compose down
docker-compose up -d
```

### Database Connection Failed

**Solution:**
```bash
# Verify postgres is healthy
docker-compose ps

# Check postgres logs
docker-compose logs postgres

# Test connection manually
docker-compose exec postgres psql -U appdev -d appdev

# In Application.cfc, use service name "postgres" not "localhost"
# Use internal port 5432, not external port 15432
```

---

## Security Best Practices

### 1. Protect .env File

```bash
# Add to .gitignore
echo ".env" >> .gitignore
echo "lucee-config/" >> .gitignore
echo "data/" >> .gitignore

# Create example file for version control
cp .env .env.example

# Edit .env.example and replace sensitive values
nano .env.example
# Change: LUCEE_ADMIN_PASSWORD=your_password_here
```

### 2. Use Strong Passwords

Update `.env` with strong passwords:
```bash
LUCEE_ADMIN_PASSWORD=$(openssl rand -base64 24)
POSTGRES_PASSWORD=$(openssl rand -base64 24)
```

### 3. Restrict File Permissions

```bash
chmod 600 .env
chmod -R 750 lucee-config/
```

---

## Migration Checklist

Use this checklist for each server:

- [ ] Document current container setup
- [ ] Create project directory structure
- [ ] Backup Lucee configurations
- [ ] Backup PostgreSQL database
- [ ] Create .env file with correct paths
- [ ] Create docker-compose.yml
- [ ] Validate configuration (`docker-compose config`)
- [ ] Stop legacy containers
- [ ] Start services (`docker-compose up -d`)
- [ ] Verify container status (`docker-compose ps`)
- [ ] Test Lucee Admin access
- [ ] Test PostgreSQL connection
- [ ] Test application functionality
- [ ] Configure datasources if needed
- [ ] Update documentation
- [ ] Remove legacy containers (`docker rm`)

---

## Additional Services (Optional)

### Add pgAdmin (Web-based PostgreSQL Manager)

Add to `docker-compose.yml`:

```yaml
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: lucee-pgadmin
    ports:
      - "5050:80"
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@example.com
      - PGADMIN_DEFAULT_PASSWORD=admin123
    volumes:
      - pgadmin-data:/var/lib/pgadmin
    restart: unless-stopped
    networks:
      - lucee-network

volumes:
  pgadmin-data:
```

Add to `.env`:
```bash
PGADMIN_EMAIL=admin@example.com
PGADMIN_PASSWORD=admin123
```

Access: `http://localhost:5050`

### Add Redis (Caching)

```yaml
  redis:
    image: redis:alpine
    container_name: lucee-redis
    ports:
      - "6379:6379"
    restart: unless-stopped
    networks:
      - lucee-network
```

---

## Backup and Restore Procedures

### Backup

```bash
# Full PostgreSQL backup
docker-compose exec postgres pg_dumpall -U appdev > backup-full-$(date +%Y%m%d-%H%M%S).sql

# Specific database backup
docker-compose exec postgres pg_dump -U appdev appdev > backup-appdev-$(date +%Y%m%d-%H%M%S).sql

# Backup Lucee configs
tar -czf lucee-config-backup-$(date +%Y%m%d-%H%M%S).tar.gz lucee-config/

# Backup application code
tar -czf www-backup-$(date +%Y%m%d-%H%M%S).tar.gz www/
```

### Restore

```bash
# Restore PostgreSQL database
docker-compose exec -T postgres psql -U appdev -d appdev < backup-appdev-20251231.sql

# Restore Lucee configs
tar -xzf lucee-config-backup-20251231.tar.gz

# Restart services
docker-compose restart
```

---

## Environment-Specific Configurations

Create multiple .env files for different environments:

### .env.development
```bash
LUCEE_PORT=8889
POSTGRES_PORT=15432
APP_PATH=/Users/username/Documents/appdev-dev/www
```

### .env.staging
```bash
LUCEE_PORT=8890
POSTGRES_PORT=15433
APP_PATH=/Users/username/Documents/appdev-staging/www
```

### .env.production
```bash
LUCEE_PORT=8891
POSTGRES_PORT=15434
APP_PATH=/Users/username/Documents/appdev-prod/www
```

**Usage:**
```bash
# Start with specific environment
docker-compose --env-file .env.development up -d
docker-compose --env-file .env.staging up -d
docker-compose --env-file .env.production up -d
```

---

## Notes

- Always backup before migrating
- Test in a non-production environment first
- Keep `.env` file secure and out of version control
- Document any custom configurations
- Update this guide with lessons learned

---

## Quick Reference Card

```bash
# Daily Operations
docker-compose up -d              # Start services
docker-compose down               # Stop services
docker-compose logs -f            # View logs
docker-compose ps                 # Check status
docker-compose restart            # Restart all

# Maintenance
docker-compose pull               # Update images
docker-compose exec mylucee bash  # Access container
docker-compose exec postgres psql -U appdev -d appdev

# Troubleshooting
docker-compose logs mylucee       # View Lucee logs
docker-compose logs postgres      # View DB logs
docker-compose config             # Validate config
```

---

## Support and Resources

- Lucee Documentation: https://docs.lucee.org/
- Docker Compose Documentation: https://docs.docker.com/compose/
- PostgreSQL Documentation: https://www.postgresql.org/docs/

---

**End of Migration Guide**

**Document Version**: 1.0  
**Last Updated**: December 31, 2025  
**Maintainer**: Kwame (Akimtech, LLC)
