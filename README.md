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

### 1. Create a Project Directory

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
mkdir -p plugins logs uploads
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
| `./plugins` | `/app/plugins` | Product lookup plugins and their configuration. Place plugin DLLs and `config.json` here. |
| `./logs` | `/app/logs` | Application log files for debugging and monitoring. |
| `./uploads` | `/app/wwwroot/uploads` | User-uploaded files such as recipe images and product photos. |
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

Back up these directories:
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

