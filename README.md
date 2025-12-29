# Famick Home Management (Self-Hosted)

Open-source household management system for inventory tracking, recipe management, chores, task organization, and more.

## Features

- **Stock Management** - Track household inventory with barcode scanning
- **Shopping Lists** - Plan purchases and share lists with household members
- **Recipes** - Store recipes with ingredient linking and meal planning
- **Chores** - Schedule and track recurring household tasks
- **Tasks** - Personal task management and to-do lists
- **Plugin System** - Extensible product lookup via plugins (e.g., USDA FoodData)

## Requirements

- Docker 20.10+
- Docker Compose 2.0+
- 2GB RAM minimum
- 10GB disk space

## Getting Started

### 1. Create the Application Directory

On Ubuntu/Debian servers, the recommended location is `/opt/homemanagement`:

```bash
sudo mkdir -p /opt/homemanagement
sudo chown -R $USER:$USER /opt/homemanagement
cd /opt/homemanagement
```

For local development or other systems:

```bash
mkdir homemanagement && cd homemanagement
```

### 2. Download the Docker Compose File

Download the example `docker-compose.yml` from this repository or create one with the following content:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: homemanagement-db
    environment:
      POSTGRES_DB: homemanagement
      POSTGRES_USER: homemanagement
      POSTGRES_PASSWORD: ${DB_PASSWORD:-changeme123}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U homemanagement"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - homemanagement-network

  web:
    image: famick/homemanagement:latest
    container_name: homemanagement-web
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:80
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=homemanagement;Username=homemanagement;Password=${DB_PASSWORD:-changeme123}
      - SelfHosted__TenantId=00000000-0000-0000-0000-000000000001
      - SelfHosted__ApplicationName=Home Management
      - JwtSettings__SecretKey=${JWT_SECRET_KEY:-your-secret-key-change-this-min-32-characters-long}
      - JwtSettings__Issuer=http://localhost
      - JwtSettings__Audience=http://localhost
      - JwtSettings__AccessTokenExpirationMinutes=15
      - JwtSettings__RefreshTokenExpirationDays=7
    ports:
      - "5000:80"
    volumes:
      - ./plugins:/app/plugins
      - ./logs:/app/logs
      - ./uploads:/app/wwwroot/uploads
    depends_on:
      postgres:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - homemanagement-network

networks:
  homemanagement-network:
    driver: bridge

volumes:
  postgres_data:
    driver: local
```

### 3. Create Environment File

Create a `.env` file with your configuration:

```bash
# Database Configuration
DB_PASSWORD=your_secure_database_password

# JWT Authentication (REQUIRED: change to a random string of at least 32 characters)
JWT_SECRET_KEY=your-random-secret-key-at-least-32-characters-long
```

### 4. Create Volume Directories

```bash
mkdir -p plugins logs uploads certs keys
```

Your directory structure should look like:

```
/opt/homemanagement/
├── docker-compose.yml
├── .env
├── plugins/
├── logs/
├── uploads/
├── certs/
└── keys/
```

### 5. Start the Application

```bash
docker compose up -d
```

### 6. Access the Application

Open your browser and navigate to `http://localhost:5000`

On first access, you'll be prompted to create an admin account and configure your household.

## Volume Mounts

The application uses several volume mounts for persistent data:

| Host Path | Container Path | Purpose |
|-----------|----------------|---------|
| `./config` | `/app/config` | Configuration overrides (appsettings.json). See [Configuration File](#configuration-file). |
| `./plugins` | `/app/plugins` | Product lookup plugins and their configuration. Place plugin DLLs and `config.json` here. |
| `./logs` | `/app/logs` | Application log files for debugging and monitoring. |
| `./uploads` | `/app/wwwroot/uploads` | User-uploaded files such as recipe images and product photos. |
| `./keys` | `/root/.aspnet/DataProtection-Keys` | Encryption keys for auth cookies. Persists user sessions across restarts. |
| `postgres_data` | `/var/lib/postgresql/data` | PostgreSQL database storage (Docker named volume). |

### Plugins Directory

The plugins directory supports:
- **Built-in plugins** - Configured via `config.json`
- **External plugins** - Custom DLLs in `external/` subdirectory

Example `plugins/config.json`:
```json
[
  {
    "id": "usda",
    "enabled": true,
    "builtin": true,
    "displayName": "USDA FoodData Central",
    "config": {
      "apiKey": "YOUR_USDA_API_KEY"
    }
  }
]
```

## Configuration

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `DB_PASSWORD` | PostgreSQL password | `changeme123` |
| `JWT_SECRET_KEY` | JWT signing key (min 32 chars) | - |
| `ASPNETCORE_ENVIRONMENT` | Runtime environment | `Production` |
| `JwtSettings__AccessTokenExpirationMinutes` | Access token lifetime | `15` |
| `JwtSettings__RefreshTokenExpirationDays` | Refresh token lifetime | `7` |

### HTTPS Configuration

For production deployments with HTTPS, add certificate configuration:

```yaml
environment:
  - ASPNETCORE_URLS=http://+:80;https://+:443
  - ASPNETCORE_Kestrel__Certificates__Default__Path=/https/aspnetapp.pfx
  - ASPNETCORE_Kestrel__Certificates__Default__Password=${CERT_PASSWORD}
ports:
  - "5000:80"
  - "5001:443"
volumes:
  - ./certs:/https:ro
```

Place your `.pfx` certificate file in the `./certs` directory.

### Configuration File

You can override application settings by creating `config/appsettings.json`:

```bash
mkdir -p config
```

Example `config/appsettings.json`:
```json
{
  "ReverseProxy": {
    "TrustedNetworks": ["172.16.0.0/12", "10.0.0.0/8"]
  }
}
```

This file is mounted read-only and takes precedence over default settings.

### Reverse Proxy Configuration

When running behind nginx or another reverse proxy, the application automatically handles forwarded headers. By default, it trusts all proxies (suitable for Docker environments).

For stricter security, configure trusted proxies in `config/appsettings.json`:

```json
{
  "ReverseProxy": {
    "TrustedProxies": ["192.168.1.100", "10.0.0.1"],
    "TrustedNetworks": ["172.16.0.0/12"]
  }
}
```

| Setting | Description | Example |
|---------|-------------|---------|
| `TrustedProxies` | Specific proxy IP addresses | `["192.168.1.100"]` |
| `TrustedNetworks` | CIDR ranges to trust | `["172.16.0.0/12", "10.0.0.0/8"]` |

Common network ranges:
- Docker bridge: `172.17.0.0/16`
- Docker compose: `172.16.0.0/12`
- Kubernetes: `10.0.0.0/8`

Example nginx configuration:
```nginx
location / {
    proxy_pass http://homemanagement-web:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
}
```

#### Nginx Proxy Manager

If using [Nginx Proxy Manager](https://nginxproxymanager.com/):

1. Add a new Proxy Host
2. Set the **Domain Names** to your domain
3. Set **Forward Hostname/IP** to the container name or IP (e.g., `homemanagement-web`)
4. Set **Forward Port** to `80`
5. Enable **Block Common Exploits** and **Websockets Support**
6. Configure SSL under the **SSL** tab (e.g., Let's Encrypt)

The default configuration works out of the box. For custom headers, use the **Custom locations** tab (not the Advanced tab) and add a location for `/`.

#### Diagnostic Endpoint

To verify your proxy configuration is working correctly, access:
```
https://your-domain/api/setup/diagnostics
```

This returns information about how the application sees the incoming request, including forwarded headers, detected scheme, and client IP.

## Optional: Database Administration

To include pgAdmin for database management:

```bash
docker compose --profile tools up -d
```

Access pgAdmin at `http://localhost:5050` with:
- Email: `admin@local.dev`
- Password: `admin`

## Updating

To update to the latest version:

```bash
docker compose pull
docker compose up -d
```

## Backup

### Database Backup

```bash
docker exec homemanagement-db pg_dump -U homemanagement homemanagement > backup.sql
```

### Database Restore

```bash
docker exec -i homemanagement-db psql -U homemanagement homemanagement < backup.sql
```

### Full Backup

Back up these directories (from `/opt/homemanagement/`):

```bash
tar -czvf homemanagement-backup.tar.gz plugins uploads
```

Or individually:
- `./plugins` - Plugin configuration
- `./uploads` - User uploads
- Database (see above)

## License

This project is licensed under the **PolyForm Shield License 1.0.0** - see [LICENSE](LICENSE) for details.

- Free to use for any purpose that does not compete with Famick products
- Modify and distribute with attribution
- Self-hosting for personal/business use allowed
- Cannot be used to create competing products or services

## Support

- Bug Reports: [GitHub Issues](https://github.com/famick/homemanagement/issues)
- Discussions: [GitHub Discussions](https://github.com/famick/homemanagement/discussions)

