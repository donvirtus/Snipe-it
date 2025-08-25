# Snipe-IT Installation Guide Using Docker

This guide provides a detailed step-by-step process to install and configure Snipe-IT, an open-source IT asset management system, using Docker. It includes all necessary commands, explanations, troubleshooting tips, and usage instructions.

## Prerequisites

- **Operating System**: Linux (e.g., Ubuntu 24.04), macOS, or Windows with WSL2.
- **Docker**: Installed and running (version 20.10 or higher).
- **Docker Compose**: Installed (version 1.29 or higher).
- **Internet Connection**: Required for downloading Docker images.
- **Basic Terminal Knowledge**: Familiarity with command-line operations.

### Installation of Prerequisites
1. Install Docker:
   ```bash
   sudo apt update && sudo apt install -y docker.io
   sudo systemctl start docker
   sudo systemctl enable docker
   ```
   - **Explanation**: Updates package list, installs Docker, starts the service, and enables it to run on boot.

2. Install Docker Compose:
   ```bash
   sudo curl -L "https://github.com/docker/compose/releases/download/v2.24.6/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   ```
   - **Explanation**: Downloads the latest Docker Compose binary and makes it executable.

3. Add your user to the `docker` group (to run Docker without `sudo`):
   ```bash
   sudo usermod -aG docker $USER
   newgrp docker
   ```
   - **Explanation**: Grants your user permission to manage Docker. Log out and back in after this step for changes to take effect.

## Setup Instructions

### 1. Create Project Directory
```bash
mkdir snipe_it && cd snipe_it
```
- **Explanation**: Creates a directory named `snipe_it` and navigates into it to store all configuration files.

### 2. Create Required Files
- Create `docker-compose.yml`:
  ```yaml
  # Compose file for production.
  
  volumes:
    db_data:
    storage:

  services:
    app:
      image: snipe/snipe-it:${APP_VERSION:-latest}
      restart: unless-stopped
      volumes:
        - storage:/var/lib/snipeit
      ports:
        - "${APP_PORT:-8000}:80"
      depends_on:
        db:
          condition: service_healthy
      env_file:
        - .env

    db:
      image: mariadb:11.4.7
      restart: unless-stopped
      volumes:
        - db_data:/var/lib/mysql
      environment:
        MYSQL_DATABASE: ${DB_DATABASE}
        MYSQL_USER: ${DB_USERNAME}
        MYSQL_PASSWORD: ${DB_PASSWORD}
        MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      healthcheck:
        test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
        interval: 5s
        timeout: 1s
        retries: 5
      ports:
        - "3307:3306"  # Use 3307 on host to avoid conflict with local MariaDB
  ```
  - **Explanation**: Defines two services (`app` for Snipe-IT and `db` for MariaDB). The `ports` mapping for `db` uses 3307 on the host to avoid conflicts with a local MariaDB instance.

- Create `.env` file:
  ```env
  # REQUIRED: DOCKER SPECIFIC SETTINGS
  APP_VERSION=
  APP_PORT=8000

  # REQUIRED: BASIC APP SETTINGS
  APP_ENV=production
  APP_DEBUG=false
  APP_KEY=base64:3ilviXqB9u6DX1NRcyWGJ+sjySF+H18CPDGb3+IVwMQ=
  APP_URL=http://<YOUR_IP>:8000
  APP_TIMEZONE='UTC'
  APP_LOCALE=en-US
  MAX_RESULTS=500

  # REQUIRED: DATABASE SETTINGS
  DB_CONNECTION=mysql
  DB_HOST=db
  DB_PORT='3306'
  DB_DATABASE=snipeit_db
  DB_USERNAME=snipeit_user
  DB_PASSWORD=S3k4r3pmu!
  MYSQL_ROOT_PASSWORD=S3k4r3pmu!
  DB_PREFIX=null
  DB_CHARSET=utf8mb4
  DB_COLLATION=utf8mb4_unicode_ci

  # REQUIRED: OUTGOING MAIL SERVER SETTINGS
  MAIL_MAILER=smtp
  MAIL_HOST=mailhog
  MAIL_PORT=1025
  MAIL_USERNAME=null
  MAIL_PASSWORD=null
  MAIL_FROM_ADDR=you@example.com
  MAIL_FROM_NAME='Snipe-IT'

  # REQUIRED: IMAGE LIBRARY
  IMAGE_LIB=gd
  ```
  - **Explanation**: Configures environment variables for Snipe-IT and MariaDB. Replace `<YOUR_IP>` with your server's IP (e.g., `20.1.1.9`). Generate a new `APP_KEY` with `docker compose run --rm app php artisan key:generate --show` if needed.

### 3. Start the Services
```bash
docker compose up -d
```
- **Explanation**: Builds and starts the containers in detached mode. The `-d` flag runs them in the background.

### 4. Verify Container Status
```bash
docker compose ps -a
```
- **Explanation**: Checks the status of all containers. Ensure both `snipe_it-app-1` and `snipe_it-db-1` are `Up` and `healthy`.

### 5. Complete the Setup Wizard
- Open a browser and navigate to `http://<YOUR_IP>:8000/setup`.
- Follow the wizard:
  - Enter database details (pre-filled from `.env`: host `db`, database `snipeit_db`, user `snipeit_user`, password `S3k4r3pmu!`).
  - Create an admin user.
  - Submit to finalize installation.
- **Explanation**: The wizard configures Snipe-IT and migrates the database. If it fails, see troubleshooting below.

### 6. Access Snipe-IT
- Visit `http://<YOUR_IP>:8000` and log in with the admin credentials created.

## Troubleshooting

- **Error: "address already in use" on port 3306**
  - **Cause**: A local MariaDB/MySQL instance is using port 3306.
  - **Solution**: 
    1. Check for local MariaDB:
       ```bash
       sudo netstat -tuln | grep 3306
       ```
    2. Stop it if not needed:
       ```bash
       sudo systemctl stop mariadb
       ```
    3. Or change the port in `docker-compose.yml` to `3307:3306` and restart:
       ```bash
       docker compose down
       docker compose up -d
       ```

- **Wizard Fails to Connect to Database**
  - **Cause**: Timing issue or cache problem.
  - **Solution**:
    1. Check database logs:
       ```bash
       docker compose logs db
       ```
    2. Clear cache:
       ```bash
       docker compose exec app php artisan cache:clear
       ```
    3. Restart containers:
       ```bash
       docker compose restart
       ```

- **Connection Refused on Browser**
  - **Cause**: Incorrect `APP_URL` or port mismatch.
  - **Solution**: Ensure `APP_URL` in `.env` matches the exposed port (e.g., `http://20.1.1.9:8000`) and restart containers.

## Usage Instructions

### Managing Containers
- **Stop Containers**:
  ```bash
  docker compose stop
  ```
  - **Explanation**: Stops the containers without removing them.

- **Restart Containers**:
  ```bash
  docker compose restart
  ```
  - **Explanation**: Restarts the containers with current configurations.

- **Remove Containers and Volumes**:
  ```bash
  docker compose down -v
  ```
  - **Explanation**: Stops and removes containers, networks, and volumes (data will be lost unless backed up).

### Accessing Database with DBeaver
1. Configure a new MariaDB connection:
   - Host: `<YOUR_IP>` (e.g., `20.1.1.9`)
   - Port: `3307` (or 3306 if local MariaDB is stopped)
   - Database: `snipeit_db`
   - Username: `snipeit_user`
   - Password: `S3k4r3pmu!`
2. Test and connect to manage the database.

### Updating Snipe-IT
1. Pull the latest image:
   ```bash
   docker compose pull
   ```
2. Restart containers:
   ```bash
   docker compose up -d
   ```
3. Run migrations:
   ```bash
   docker compose exec app php artisan migrate
   ```

## Additional Notes

- **Security**: Do not expose port 3307 publicly in production. Use SSH tunneling or a VPN for database access.
- **Backup**: Regularly back up the `db_data` and `storage` volumes:
  ```bash
  docker compose exec db mysqldump -u root -p${MYSQL_ROOT_PASSWORD} snipeit_db > backup.sql
  ```
- **Custom Ports**: If port 8000 or 3307 is in use, adjust `APP_PORT` in `.env` and `ports` in `docker-compose.yml` accordingly.
- **Logs**: Check application or database logs for debugging:
  ```bash
  docker compose logs app
  docker compose logs db
  ```

## Contributing
Feel free to suggest improvements to this guide by sharing your experience or additional troubleshooting tips.
