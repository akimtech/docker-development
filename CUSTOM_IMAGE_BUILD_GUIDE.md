# Custom Lucee Docker Image Build Guide
## Building Your Own Lucee Image with Custom Packages

**Author**: Kwame (Akimtech, LLC)  
**Date**: January 14, 2026  
**Purpose**: Step-by-step guide to create and use custom Lucee Docker images

---

## Table of Contents

1. [Overview](#overview)
2. [When to Build a Custom Image](#when-to-build-a-custom-image)
3. [Prerequisites](#prerequisites)
4. [Understanding the Dockerfile](#understanding-the-dockerfile)
5. [Step-by-Step Build Process](#step-by-step-build-process)
6. [Common Customizations](#common-customizations)
7. [Building and Testing](#building-and-testing)
8. [Using with Docker Compose](#using-with-docker-compose)
9. [Troubleshooting](#troubleshooting)
10. [Advanced Topics](#advanced-topics)

---

## Overview

This guide shows you how to create a custom Docker image based on the official Lucee image. You'll learn to add custom packages, configure system settings, and integrate your custom image with Docker Compose.

**What You'll Build:**
- A Dockerfile that extends `lucee/lucee:latest`
- Custom packages installed (e.g., Tesseract OCR, ImageMagick, etc.)
- System configurations and optimizations
- Integration with your existing Docker Compose setup

---

## When to Build a Custom Image

Build a custom image when you need:

✅ **Additional System Packages**
- OCR tools (Tesseract)
- Image processing libraries (ImageMagick, GraphicsMagick)
- PDF utilities (pdftk, ghostscript)
- Command-line tools (curl, wget, git)

✅ **Custom Configurations**
- Modified JVM settings
- Pre-installed Lucee extensions
- Custom security settings
- Performance optimizations

✅ **Reproducible Environments**
- Consistent development/production environments
- Version-controlled infrastructure
- Easy team onboarding

❌ **Don't build a custom image for:**
- Application code (use volumes instead)
- Environment-specific configs (use .env files)
- Temporary testing (use exec commands instead)

---

## Prerequisites

Before starting, ensure you have:

- ✅ Docker Desktop installed and running
- ✅ Basic understanding of Docker concepts
- ✅ A working Docker Compose setup (from Migration Guide)
- ✅ Text editor (nano, vim, VS Code, etc.)
- ✅ Terminal/command line access

**Verify Prerequisites:**

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker-compose --version

# Verify Docker is running
docker ps
```

---

## Understanding the Dockerfile

### What is a Dockerfile?

A Dockerfile is a text file containing instructions to build a Docker image. Think of it as a recipe:

- **FROM**: The base image you're starting with
- **RUN**: Commands to execute during build (install packages, etc.)
- **ENV**: Set environment variables
- **COPY/ADD**: Add files from your computer into the image
- **WORKDIR**: Set the working directory
- **EXPOSE**: Document which ports the container uses
- **CMD**: Default command when container starts

### Basic Structure

```dockerfile
# Start from base image
FROM lucee/lucee:latest

# Install packages
RUN apt-get update && apt-get install -y package-name

# Set environment variables
ENV MY_VAR=value

# Copy files
COPY myfile.txt /destination/

# Set working directory
WORKDIR /var/www

# Default command (usually inherited from base image)
CMD ["catalina.sh", "run"]
```

---

## Step-by-Step Build Process

### Step 1: Create Your Dockerfile

Navigate to your project directory and create a Dockerfile:

```bash
# Navigate to your project directory
cd /Users/username/Documents/appdev

# Create Dockerfile
nano Dockerfile
```

### Step 2: Basic Dockerfile Template

Start with this basic template:

```dockerfile
# Use the official Lucee image as base
FROM lucee/lucee:latest

# Metadata
LABEL maintainer="your-email@example.com"
LABEL description="Custom Lucee image with additional packages"
LABEL version="1.0"

# Switch to root user to install packages
USER root

# Update package lists and install basic utilities
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    vim \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Switch back to lucee user (important for security)
USER lucee

# Expose port 8888 (already exposed in base image, but good to document)
EXPOSE 8888

# The CMD is inherited from the base image
```

Save and exit: `Ctrl+X`, then `Y`, then `Enter`

### Step 3: Understanding the Build Context

When you build an image, Docker uses a "build context" - all files in the directory where the Dockerfile is located.

**Best Practice Directory Structure:**

```
/Users/username/Documents/appdev/
├── Dockerfile                    # Your custom image definition
├── docker-compose.yml            # Container orchestration
├── .env                          # Environment variables
├── .dockerignore                 # Files to exclude from build
├── lucee-config/
│   ├── server/
│   └── web/
├── www/                          # Application code (excluded from build)
└── data/                         # Database data (excluded from build)
```

### Step 4: Create .dockerignore File

Exclude unnecessary files from the build context to speed up builds:

```bash
nano .dockerignore
```

**Template .dockerignore:**

```
# Exclude application code (mounted as volume)
www/*

# Exclude database data
data/*

# Exclude config directories (mounted as volumes)
lucee-config/*

# Exclude backups
*.sql
*.tar.gz
lucee-backup/*

# Exclude environment files
.env
.env.*

# Exclude version control
.git
.gitignore

# Exclude documentation
*.md
README*

# Exclude IDE files
.vscode/
.idea/
*.swp
*.swo

# Exclude macOS files
.DS_Store

# Exclude logs
*.log
```

Save and exit: `Ctrl+X`, then `Y`, then `Enter`

---

## Common Customizations

### Example 1: Adding Tesseract OCR

Perfect for reading text from images and PDFs:

```dockerfile
FROM lucee/lucee:latest

LABEL maintainer="kwame@akimtech.com"
LABEL description="Lucee with Tesseract OCR for document processing"

USER root

# Install Tesseract OCR and language packs
RUN apt-get update && apt-get install -y \
    tesseract-ocr \
    tesseract-ocr-eng \
    tesseract-ocr-spa \
    tesseract-ocr-fra \
    libtesseract-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Verify installation
RUN tesseract --version

USER lucee

EXPOSE 8888
```

### Example 2: Adding ImageMagick

For advanced image processing:

```dockerfile
FROM lucee/lucee:latest

LABEL maintainer="kwame@akimtech.com"
LABEL description="Lucee with ImageMagick for image processing"

USER root

# Install ImageMagick and dependencies
RUN apt-get update && apt-get install -y \
    imagemagick \
    libmagickwand-dev \
    ghostscript \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Verify installation
RUN convert --version

USER lucee

EXPOSE 8888
```

### Example 3: Adding PDF Tools

For PDF manipulation:

```dockerfile
FROM lucee/lucee:latest

LABEL maintainer="kwame@akimtech.com"
LABEL description="Lucee with PDF processing tools"

USER root

# Install PDF utilities
RUN apt-get update && apt-get install -y \
    poppler-utils \
    pdftk \
    ghostscript \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Verify installations
RUN pdfinfo -v && pdftk --version

USER lucee

EXPOSE 8888
```

### Example 4: Complete Custom Image (Kitchen Sink)

Combines multiple tools - use this as your starting point:

```dockerfile
FROM lucee/lucee:latest

LABEL maintainer="kwame@akimtech.com"
LABEL description="Custom Lucee image with OCR, image processing, and PDF tools"
LABEL version="1.0"

USER root

# Install all the things!
RUN apt-get update && apt-get install -y \
    # OCR Tools
    tesseract-ocr \
    tesseract-ocr-eng \
    libtesseract-dev \
    # Image Processing
    imagemagick \
    libmagickwand-dev \
    graphicsmagick \
    # PDF Tools
    poppler-utils \
    ghostscript \
    # Utilities
    curl \
    wget \
    vim \
    git \
    unzip \
    # Development tools
    build-essential \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set ImageMagick security policy (allow PDF processing)
RUN sed -i 's/rights="none" pattern="PDF"/rights="read|write" pattern="PDF"/' \
    /etc/ImageMagick-6/policy.xml

# Set environment variables
ENV TESSDATA_PREFIX=/usr/share/tesseract-ocr/4.00/tessdata

# Create working directories
RUN mkdir -p /tmp/lucee-temp && \
    chown -R lucee:lucee /tmp/lucee-temp

USER lucee

EXPOSE 8888
```

### Example 5: Adding Java Options

Customize JVM settings for better performance:

```dockerfile
FROM lucee/lucee:latest

LABEL maintainer="kwame@akimtech.com"
LABEL description="Lucee with custom JVM settings"

USER root

# Set Java options
ENV JAVA_OPTS="-Xms512m -Xmx2048m -XX:+UseG1GC -Djava.awt.headless=true"

# Install monitoring tools
RUN apt-get update && apt-get install -y \
    htop \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER lucee

EXPOSE 8888
```

---

## Building and Testing

### Step 1: Build Your Custom Image

From your project directory:

```bash
# Navigate to project directory
cd /Users/username/Documents/appdev

# Build the image (replace 'my-lucee' with your preferred name)
docker build -t my-custom-lucee:latest .

# The dot (.) at the end is important - it means "current directory"
```

**Understanding the Build Command:**

- `docker build`: The build command
- `-t my-custom-lucee:latest`: Tag the image with a name and version
- `.`: Use the current directory as build context

**Build Process Output:**

You'll see output like:

```
[+] Building 45.3s (12/12) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [1/7] FROM docker.io/lucee/lucee:latest
 => [2/7] RUN apt-get update && apt-get install -y tesseract-ocr
 => [3/7] RUN tesseract --version
 => exporting to image
 => => naming to docker.io/library/my-custom-lucee:latest
```

### Step 2: Verify the Build

Check that your image was created:

```bash
# List all images
docker images

# Look for your image
docker images | grep my-custom-lucee
```

**Expected Output:**

```
REPOSITORY          TAG       IMAGE ID       CREATED         SIZE
my-custom-lucee     latest    abc123def456   2 minutes ago   1.2GB
```

### Step 3: Test the Image

Test your custom image before using it with Docker Compose:

```bash
# Run a test container
docker run -d \
  --name test-lucee \
  -p 9999:8888 \
  -e LUCEE_ADMIN_PASSWORD=test123 \
  my-custom-lucee:latest

# Check if it's running
docker ps | grep test-lucee

# View logs
docker logs test-lucee

# Test in browser
open http://localhost:9999
```

### Step 4: Test Installed Packages

Verify that your custom packages are installed:

```bash
# Access the container shell
docker exec -it test-lucee bash

# Inside the container, test your installations:
tesseract --version        # If you installed Tesseract
convert --version          # If you installed ImageMagick
pdfinfo -v                 # If you installed poppler-utils

# Exit the container
exit
```

### Step 5: Clean Up Test Container

Once testing is complete:

```bash
# Stop and remove test container
docker stop test-lucee
docker rm test-lucee
```

---

## Using with Docker Compose

### Step 1: Update docker-compose.yml

Modify your `docker-compose.yml` to use your custom image:

```bash
nano docker-compose.yml
```

**Replace the Lucee service configuration:**

```yaml
version: '3.8'

services:
  mylucee:
    build:
      context: .                    # Build from current directory
      dockerfile: Dockerfile        # Use this Dockerfile
    image: my-custom-lucee:latest   # Tag the built image
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

### Step 2: Build and Start Services

```bash
# Stop any running containers first
docker-compose down

# Build the image and start services
docker-compose up -d --build

# The --build flag forces a rebuild of the image
```

### Step 3: Verify Everything Works

```bash
# Check container status
docker-compose ps

# View logs
docker-compose logs -f mylucee

# Test in browser
open http://localhost:8889
```

---

## Dockerfile Best Practices

### 1. Layer Optimization

Combine commands to reduce layers:

**❌ Bad (creates multiple layers):**

```dockerfile
RUN apt-get update
RUN apt-get install -y package1
RUN apt-get install -y package2
RUN apt-get clean
```

**✅ Good (single layer):**

```dockerfile
RUN apt-get update && apt-get install -y \
    package1 \
    package2 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### 2. Use Specific Package Versions

For reproducible builds:

```dockerfile
# Instead of:
RUN apt-get install -y tesseract-ocr

# Use:
RUN apt-get install -y tesseract-ocr=4.1.1-2.1
```

### 3. Clean Up After Installing

Always clean up to reduce image size:

```dockerfile
RUN apt-get update && apt-get install -y \
    package-name \
    # Clean up
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/*
```

### 4. Use .dockerignore

Always create a `.dockerignore` file to exclude:
- Application code (mounted as volumes)
- Database data
- Environment files
- Documentation
- Version control files

### 5. Security Best Practices

```dockerfile
# Don't run as root
USER lucee

# Use specific base image versions
FROM lucee/lucee:5.4.0.48

# Don't expose unnecessary ports
EXPOSE 8888

# Don't include secrets in the image
# Use environment variables instead
```

---

## Build Commands Reference

### Basic Build Commands

```bash
# Build with tag
docker build -t my-image:latest .

# Build with custom Dockerfile name
docker build -f Dockerfile.custom -t my-image:latest .

# Build without cache (fresh build)
docker build --no-cache -t my-image:latest .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t my-image:latest .
```

### Build with Docker Compose

```bash
# Build images defined in docker-compose.yml
docker-compose build

# Build specific service
docker-compose build mylucee

# Build without cache
docker-compose build --no-cache

# Build and start containers
docker-compose up -d --build

# Build with pull (get latest base image)
docker-compose build --pull
```

### Image Management

```bash
# List images
docker images

# Remove image
docker rmi my-custom-lucee:latest

# Remove unused images
docker image prune

# Remove all unused images
docker image prune -a

# Tag an image
docker tag my-custom-lucee:latest my-custom-lucee:v1.0

# Push to Docker Hub (if you have an account)
docker push username/my-custom-lucee:latest
```

---

## Troubleshooting

### Build Fails with "Package not found"

**Problem:** Package names incorrect or not available in repository

**Solution:**

```bash
# Search for the package
docker run --rm -it ubuntu:latest bash
apt-get update
apt-cache search tesseract
exit
```

### Build is Very Slow

**Problem:** Large build context or no caching

**Solutions:**

1. Create/update `.dockerignore`:
```bash
echo "data/*" >> .dockerignore
echo "www/*" >> .dockerignore
```

2. Use layer caching:
```dockerfile
# Put frequently changing commands at the end
# Put rarely changing commands at the beginning
```

### Permission Errors

**Problem:** Files created as root user

**Solution:**

```dockerfile
# Always switch back to lucee user
USER root
RUN apt-get install -y package
USER lucee
```

### Image is Too Large

**Problem:** Image size is several GB

**Solutions:**

1. Clean up in the same layer:
```dockerfile
RUN apt-get update && apt-get install -y package \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

2. Use multi-stage builds (advanced):
```dockerfile
FROM lucee/lucee:latest AS builder
# Build steps...

FROM lucee/lucee:latest
COPY --from=builder /built/artifacts /destination
```

### Docker Compose Not Using Updated Image

**Problem:** Changes to Dockerfile not reflected in containers

**Solution:**

```bash
# Force rebuild
docker-compose down
docker-compose build --no-cache mylucee
docker-compose up -d
```

---

## Advanced Topics

### Using Build Arguments

Pass variables during build time:

```dockerfile
FROM lucee/lucee:latest

# Define build argument
ARG TESSERACT_VERSION=4.1.1

USER root

RUN apt-get update && apt-get install -y \
    tesseract-ocr=${TESSERACT_VERSION}* \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

USER lucee
```

**Build with argument:**

```bash
docker build --build-arg TESSERACT_VERSION=5.0.0 -t my-lucee:latest .
```

### Multi-Stage Builds

Build tools in one stage, copy artifacts to final image:

```dockerfile
# Stage 1: Build/compile
FROM ubuntu:latest AS builder
RUN apt-get update && apt-get install -y build-essential
# Build something...

# Stage 2: Final image
FROM lucee/lucee:latest
COPY --from=builder /output /destination
```

### Health Checks in Dockerfile

```dockerfile
FROM lucee/lucee:latest

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:8888/ || exit 1
```

### Custom Entrypoint Scripts

Create initialization scripts:

```dockerfile
FROM lucee/lucee:latest

USER root

COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

USER lucee

ENTRYPOINT ["docker-entrypoint.sh"]
```

### Using Private Base Images

```dockerfile
# Use a private registry
FROM myregistry.com/lucee-custom:latest

# Or use credentials
# docker login myregistry.com
# docker build -t my-image .
```

---

## Version Control and Documentation

### Recommended Files to Track

```
project/
├── Dockerfile              # ✅ Track this
├── .dockerignore          # ✅ Track this
├── docker-compose.yml      # ✅ Track this
├── .env.example           # ✅ Track this (sanitized)
├── .env                   # ❌ Don't track (secrets)
├── README.md              # ✅ Track this
└── CHANGELOG.md           # ✅ Track this
```

### Sample .gitignore

```
# Environment files with secrets
.env
.env.local
.env.production

# Data directories
data/
lucee-config/
lucee-backup/

# Logs
*.log

# OS files
.DS_Store
```

### Documenting Your Custom Image

Create a README.md in your project:

```markdown
# My Custom Lucee Image

## Installed Packages
- Tesseract OCR 4.1.1
- ImageMagick 6.9.11
- Poppler Utils

## Build Instructions
\`\`\`bash
docker build -t my-custom-lucee:latest .
\`\`\`

## Usage
\`\`\`bash
docker-compose up -d
\`\`\`
```

---

## Deployment Strategies

### Development vs Production

**Development Dockerfile:**

```dockerfile
FROM lucee/lucee:latest

USER root

# Install dev tools
RUN apt-get update && apt-get install -y \
    tesseract-ocr \
    vim \
    htop \
    git \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Enable debugging
ENV LUCEE_DEBUG=true

USER lucee
```

**Production Dockerfile:**

```dockerfile
FROM lucee/lucee:latest

USER root

# Install only necessary packages
RUN apt-get update && apt-get install -y \
    tesseract-ocr \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Optimize for production
ENV LUCEE_DEBUG=false
ENV JAVA_OPTS="-Xms1024m -Xmx4096m"

USER lucee
```

### Tagging Strategy

```bash
# Development builds
docker build -t my-lucee:dev .

# Staging builds
docker build -t my-lucee:staging .

# Production builds with version
docker build -t my-lucee:1.0.0 .
docker build -t my-lucee:latest .
```

---

## Quick Reference

### Essential Commands

```bash
# Build image
docker build -t my-lucee:latest .

# Build with no cache
docker build --no-cache -t my-lucee:latest .

# Build with Docker Compose
docker-compose build

# Rebuild and restart
docker-compose up -d --build

# Test image
docker run -it --rm my-lucee:latest bash

# List images
docker images

# Remove image
docker rmi my-lucee:latest

# View build history
docker history my-lucee:latest

# Inspect image
docker inspect my-lucee:latest
```

---

## Next Steps

1. ✅ Create your Dockerfile based on templates above
2. ✅ Add packages you need (Tesseract, ImageMagick, etc.)
3. ✅ Build and test your image
4. ✅ Update docker-compose.yml to use your custom image
5. ✅ Document your customizations
6. ✅ Version control your Dockerfile
7. ✅ Create different images for dev/staging/prod if needed

---

## Support and Resources

- **Docker Documentation**: https://docs.docker.com/engine/reference/builder/
- **Lucee Docker Hub**: https://hub.docker.com/r/lucee/lucee
- **Dockerfile Best Practices**: https://docs.docker.com/develop/dev-best-practices/
- **Docker Compose Documentation**: https://docs.docker.com/compose/

---

**End of Custom Image Build Guide**

**Document Version**: 1.0  
**Last Updated**: January 14, 2026  
**Maintainer**: Kwame (Akimtech, LLC)
