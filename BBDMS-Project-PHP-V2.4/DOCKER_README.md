# Docker Setup Guide for Blood Bank Donor Management System

This guide will help you deploy the Blood Bank Donor Management System using Docker.

## Prerequisites

- Docker (version 20.10 or higher)
- Docker Compose (version 2.0 or higher)

## Quick Start

1. **Clone or navigate to the project directory**
   ```bash
   cd BBDMS-Project-PHP-V2.4
   ```

2. **Create environment file**
   ```bash
   cp env.example .env
   ```

3. **Edit the `.env` file** (optional - defaults are provided)
   - Update database credentials if needed
   - Change MySQL root password for production

4. **Start the containers**
   ```bash
   docker-compose up -d
   ```

5. **Wait for database initialization** (about 30-60 seconds)
   - The database will be automatically created and populated with the SQL file

6. **Access the application**
   - **Main Application**: http://localhost:8080
   - **phpMyAdmin**: http://localhost:8081
   - **MySQL**: localhost:3306

## Default Credentials

### Admin Login
- **Username**: `admin`
- **Password**: `Test@123`

### Donor Login
- **Email**: `amitk@gmail.com`
- **Password**: `Test@123`

### Database Access (phpMyAdmin)
- **Server**: `db`
- **Username**: `bbdms_user` (or as set in .env)
- **Password**: `bbdms_password` (or as set in .env)

## Environment Variables

The following environment variables can be configured in the `.env` file:

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | `db` | Database hostname (use `db` for Docker) |
| `DB_USER` | `bbdms_user` | Database username |
| `DB_PASS` | `bbdms_password` | Database password |
| `DB_NAME` | `bbdms` | Database name |
| `MYSQL_ROOT_PASSWORD` | `rootpassword` | MySQL root password |

## Docker Services

The `docker-compose.yml` file includes three services:

1. **web** - PHP 8.2 with Apache
   - Port: 8080
   - Application files mounted from `./bbdms`

2. **db** - MySQL 8.0
   - Port: 3306
   - Data persisted in Docker volume
   - Auto-initializes with `bbdms.sql`

3. **phpmyadmin** - phpMyAdmin interface
   - Port: 8081
   - Connected to MySQL service

## Useful Docker Commands

### Start services
```bash
docker-compose up -d
```

### Stop services
```bash
docker-compose down
```

### View logs
```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f web
docker-compose logs -f db
```

### Restart services
```bash
docker-compose restart
```

### Rebuild containers (after Dockerfile changes)
```bash
docker-compose up -d --build
```

### Access container shell
```bash
# Web container
docker-compose exec web bash

# Database container
docker-compose exec db bash
```

### Remove everything (including volumes)
```bash
docker-compose down -v
```

## Troubleshooting

### Database connection errors
- Ensure the database container is running: `docker-compose ps`
- Check database logs: `docker-compose logs db`
- Wait a few seconds after starting - MySQL needs time to initialize

### Port already in use
- Change ports in `docker-compose.yml` if 8080, 8081, or 3306 are already in use
- Update the port mappings:
  ```yaml
  ports:
    - "8082:80"  # Change 8080 to 8082
  ```

### Permission issues
- On Linux/Mac, you may need to adjust file permissions:
  ```bash
  chmod -R 755 bbdms/
  ```

### Database not initializing
- Check if the SQL file exists: `ls -la "SQL File/bbdms.sql"`
- View database logs: `docker-compose logs db`
- Manually import SQL if needed through phpMyAdmin

### Clear and restart
```bash
# Stop and remove containers, networks, and volumes
docker-compose down -v

# Remove images (optional)
docker-compose down --rmi all

# Start fresh
docker-compose up -d
```

## Production Deployment

For production deployment, consider:

1. **Change default passwords** in `.env` file
2. **Use strong passwords** for database credentials
3. **Enable HTTPS** by adding a reverse proxy (nginx/traefik)
4. **Set up regular backups** for the MySQL volume
5. **Use Docker secrets** for sensitive data
6. **Configure firewall rules** to restrict access
7. **Monitor logs** regularly

## Backup and Restore

### Backup database
```bash
docker-compose exec db mysqldump -u bbdms_user -pbbdms_password bbdms > backup.sql
```

### Restore database
```bash
docker-compose exec -T db mysql -u bbdms_user -pbbdms_password bbdms < backup.sql
```

## Volume Management

Database data is stored in a Docker volume named `bbdms_mysql_data`. To backup:

```bash
# Create backup
docker run --rm -v bbdms_mysql_data:/data -v $(pwd):/backup alpine tar czf /backup/mysql_backup.tar.gz /data

# Restore backup
docker run --rm -v bbdms_mysql_data:/data -v $(pwd):/backup alpine tar xzf /backup/mysql_backup.tar.gz -C /
```

## Support

For issues related to:
- **Docker setup**: Check this README and Docker logs
- **Application issues**: Refer to the main project README
- **Database issues**: Check MySQL logs and phpMyAdmin

## Notes

- The application files are mounted as volumes, so changes to PHP files are reflected immediately
- Database changes persist in Docker volumes even after container removal
- The SQL file is automatically imported on first database initialization
- All services are on the same Docker network (`bbdms-network`) for secure communication

