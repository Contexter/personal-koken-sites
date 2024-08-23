# Comprehensive Koken CMS Dockerization and Migration Guide

This guide provides detailed instructions for backing up your current Koken CMS installation, Dockerizing the environment, and migrating it to a new server, such as an AWS Lightsail instance. The process is automated through a shell script designed to install all necessary dependencies and ensure a smooth transition.

## 1. Backup Your Current Koken Hosting Environment

Before proceeding with the Dockerization and migration of your Koken CMS, it’s critical to back up your existing installation, including both the Koken files and the MySQL database.

### Step 1.1: Identify the Components to Back Up

Your Koken installation consists of two main components:

1. **Koken CMS Files**: These files include the core Koken system files, themes, plugins, and any media uploads (images, videos, etc.).
2. **MySQL Database**: This database stores all of your website’s content, settings, and metadata.

### Step 1.2: Backup Koken CMS Files

#### Method 1: Using FTP/SFTP
1. **Connect to Your Server**: Use an FTP or SFTP client (e.g., FileZilla, WinSCP) to connect to your current hosting server.
2. **Navigate to Koken Directory**: Locate the directory where Koken is installed. This is typically in the `public_html`, `www`, or a similarly named directory.
3. **Download the Files**: Download the entire Koken directory to your local machine. Ensure that all subdirectories, including `storage`, `themes`, and `admin`, are included.

#### Method 2: Using SSH
1. **Connect via SSH**: Use an SSH client (e.g., PuTTY, terminal) to access your server.
2. **Navigate to Koken Directory**: Use the `cd` command to navigate to the directory where Koken is installed:
   ```bash
   cd /path/to/koken_directory
   ```
3. **Create a Compressed Backup**: Use `tar` to create a compressed archive of the Koken files:
   ```bash
   tar -czvf koken_files_backup.tar.gz .
   ```
4. **Download the Backup**: Transfer the compressed backup file to your local machine using SCP or SFTP.

### Step 1.3: Backup MySQL Database

#### Method 1: Using phpMyAdmin
1. **Log In to phpMyAdmin**: Access phpMyAdmin through your hosting control panel (cPanel, Plesk, etc.).
2. **Select the Database**: Choose the database associated with your Koken installation from the list on the left.
3. **Export the Database**:
   - Click on the "Export" tab.
   - Choose the "Quick" export method and "SQL" format.
   - Click "Go" to download the database backup to your local machine.

#### Method 2: Using SSH
1. **Connect via SSH**: Use an SSH client to access your server.
2. **Export the Database**: Use `mysqldump` to export the database to a SQL file:
   ```bash
   mysqldump -u username -p database_name > koken_database_backup.sql
   ```
3. **Download the Backup**: Transfer the SQL backup file to your local machine using SCP or SFTP.

### Step 1.4: Verify Your Backups
After downloading your Koken files and MySQL database backups, verify that the backups are complete and accessible. This ensures that if anything goes wrong during the migration, you can restore your site.

## 2. Koken CMS Dockerization and Migration

Once you have secured your backups, proceed with Dockerizing your Koken CMS and migrating it to your new server environment.

### Step 2.1: Overview of the Dockerization Script

The provided shell script will automate the following tasks:

1. **Install Docker and Docker Compose**: Ensures that Docker and Docker Compose are installed on your server.
2. **Create Project Directory**: Sets up a directory for your Koken project.
3. **Generate Dockerfile**: Creates a Dockerfile that installs all necessary PHP extensions and configures Apache for Koken CMS.
4. **Generate Docker Compose Configuration**: Creates a `docker-compose.yml` file to define and manage the Koken CMS and MySQL services.
5. **Upload Koken Files**: Transfers your Koken CMS files to the server.
6. **Import MySQL Database**: Imports your MySQL database into the Dockerized MySQL container.
7. **Configure Koken**: Updates the Koken `config.php` file with the new database credentials.
8. **Optional: Set Up SSL**: Configures SSL with Let’s Encrypt for a secure HTTPS connection.
9. **Automate Backups**: Sets up a cron job for automated daily backups of your MySQL database.

### Step 2.2: The Dockerization Script

Below is the complete script that automates the Dockerization and migration of Koken CMS:

```bash
#!/bin/bash

# Welcome message
echo "Welcome to the Koken Dockerization Script."

# Function to install Docker and Docker Compose
install_docker() {
  echo "Checking if Docker is installed..."
  if ! [ -x "$(command -v docker)" ]; then
    echo "Installing Docker..."
    sudo apt-get update
    sudo apt-get install -y docker.io
    sudo systemctl start docker
    sudo systemctl enable docker
    echo "Docker installed successfully."
  else
    echo "Docker is already installed."
  fi

  echo "Checking if Docker Compose is installed..."
  if ! [ -x "$(command -v docker-compose)" ]; then
    echo "Installing Docker Compose..."
    sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
    echo "Docker Compose installed successfully."
  else
    echo "Docker Compose is already installed."
  fi
}

# Function to create project directory
create_project_directory() {
  read -p "Enter the directory name for your Koken project [default: koken-docker]: " project_dir
  project_dir=${project_dir:-koken-docker}
  if [ -d "$project_dir" ]; then
    echo "Project directory '$project_dir' already exists."
  else
    mkdir -p "$project_dir"
    echo "Project directory '$project_dir' created."
  fi
  cd "$project_dir" || exit
}

# Function to create Dockerfile with all necessary dependencies
create_dockerfile() {
  if [ -f "Dockerfile" ]; then
    echo "Dockerfile already exists."
  else
    echo "Creating Dockerfile with all necessary dependencies for Koken..."
    cat >Dockerfile <<EOL
# Use the official PHP image with Apache
FROM php:7.4-apache

# Install necessary PHP extensions and other tools
RUN apt-get update && apt-get install -y \
    libpng-dev \
    libjpeg-dev \
    libfreetype6-dev \
    libzip-dev \
    libonig-dev \
    zip \
    unzip \
    curl \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install gd mbstring mysqli pdo pdo_mysql zip json curl \
    && a2enmod rewrite

# Set the working directory
WORKDIR /var/www/html

# Expose port 80
EXPOSE 80

# Start Apache in the foreground
CMD ["apache2-foreground"]
EOL

    echo "Dockerfile created."
  fi
}

# Function to create Docker Compose file
create_docker_compose_file() {
  if [ -f "docker-compose.yml" ]; then
    echo "docker-compose.yml already exists."
  else
    echo "Creating docker-compose.yml file..."
    read -p "Enter MySQL root password: " MYSQL_ROOT_PASSWORD
    read -p "Enter Koken database name [default: koken_db]: " MYSQL_DATABASE
    MYSQL_DATABASE=${MYSQL_DATABASE:-koken_db}
    read -p "Enter Koken database user [default: koken_user]: " MYSQL_USER
    MYSQL_USER=${MYSQL_USER:-koken_user}
    read -p "Enter Koken database user password: " MYSQL_PASSWORD

    cat >docker-compose.yml <<EOL
version: '3'

services:
  web:
    build: .
    container_name: koken-web
    volumes:
      - ./koken:/var/www/html
    ports:
      - "80:80"
    restart: always
    environment:
      - APACHE_DOCUMENT_ROOT=/var/www/html

  db:
    image: mysql:5.7
    container_name: koken-db
    volumes:
      - db_data:/var/lib/mysql
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: $MYSQL_ROOT_PASSWORD
      MYSQL_DATABASE: $MYSQL_DATABASE
      MYSQL_USER: $MYSQL_USER
      MYSQL_PASSWORD: $MYSQL_PASSWORD

volumes:
  db_data:
EOL

    echo "docker-compose.yml file created."
  fi
}

# Function to upload Koken files
upload_koken_files() {
  mkdir -p koken
  if [ "$(ls -A ./koken)" ]; then
    echo "Koken files already exist in the Docker environment."
  else
    read -p "Enter the path to your local

 Koken backup directory: " local_koken_dir
    scp -r "$local_koken_dir"/* "./koken/"
    echo "Koken files uploaded to the Docker environment."
  fi
}

# Function to import the MySQL database
import_database() {
  sudo docker-compose up -d --build
  echo "Checking if the database has already been imported..."
  if sudo docker exec -i koken-db mysql -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" -e "USE $MYSQL_DATABASE;" >/dev/null 2>&1; then
    echo "Database is already imported."
  else
    read -p "Enter the path to your local Koken MySQL backup file (e.g., backup.sql): " db_backup
    sudo docker exec -i koken-db mysql -u "$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE" < "$db_backup"
    echo "Database imported successfully."
  fi
}

# Function to configure Koken
configure_koken() {
  if [ -f "koken/config.php" ]; then
    echo "Koken configuration already exists."
  else
    echo "Configuring Koken..."
    cat >koken/config.php <<EOL
<?php
define('DB_HOST', 'db');
define('DB_USER', '$MYSQL_USER');
define('DB_PASS', '$MYSQL_PASSWORD');
define('DB_NAME', '$MYSQL_DATABASE');
EOL
    echo "Koken configuration updated."
  fi
}

# Function to set up DNS and SSL (optional)
setup_dns_ssl() {
  read -p "Would you like to set up SSL with Let's Encrypt? (y/n): " setup_ssl
  if [ "$setup_ssl" == "y" ]; then
    echo "Setting up SSL with Let's Encrypt..."
    sudo docker exec -it koken-web bash -c "
    if ! [ -x \$(command -v certbot) ]; then
      apt-get update && apt-get install -y certbot python-certbot-apache
      certbot --apache
    else
      echo 'SSL is already set up.'
    fi
    "
    echo "SSL setup completed."
  else
    echo "Skipping SSL setup."
  fi
}

# Function to automate backups
automate_backups() {
  echo "Setting up automated backups..."
  read -p "Enter the path where backups should be stored: " backup_path
  if ! crontab -l | grep -q "koken_db_backup.sql"; then
    crontab -l > mycron
    echo "0 3 * * * docker exec koken-db /usr/bin/mysqldump -u $MYSQL_USER --password=$MYSQL_PASSWORD $MYSQL_DATABASE > $backup_path/koken_db_backup.sql" >> mycron
    crontab mycron
    rm mycron
    echo "Automated database backup scheduled."
  else
    echo "Automated backup is already scheduled."
  fi
}

# Main script
main() {
  install_docker
  create_project_directory
  create_dockerfile
  create_docker_compose_file
  upload_koken_files
  import_database
  configure_koken
  setup_dns_ssl
  automate_backups
  echo "Koken CMS Dockerization and migration completed."
}

# Run the main function
main
```

### Step 2.3: Usage

1. **Save the Script**: Save the provided shell script as `koken_dockerize.sh`.
2. **Make the Script Executable**:
   ```bash
   chmod +x koken_dockerize.sh
   ```
3. **Run the Script**:
   ```bash
   ./koken_dockerize.sh
   ```
4. **Follow the Prompts**: The script will prompt you for inputs such as MySQL credentials and file paths.

### Step 2.4: Idempotency

The script is designed to be idempotent, meaning it can be run multiple times without causing unintended side effects. Specifically:

- **File and Directory Checks**: The script checks for the existence of files and directories before creating or modifying them.
- **Database Import**: The script verifies whether the database has already been imported to avoid duplication.
- **Cron Job Configuration**: The script ensures that backup cron jobs are not duplicated.

### Step 2.5: Testing and Validation

Before deploying the script in a production environment, it is recommended to:

1. **Test in a Development Environment**: Run the script in a staging environment to identify any potential issues without affecting your live site.
2. **Monitor Logs**: Check Docker container logs (`docker logs koken-web`, `docker logs koken-db`) during and after setup to ensure everything is functioning as expected.
3. **Verify Functionality**: Confirm that the Koken CMS is operating correctly in the Docker environment, including database connectivity and SSL configuration.

### Step 2.6: Ongoing Maintenance

While this script is designed to automate the migration process and ensure a stable environment, regular updates and monitoring are recommended:

- **Update Docker Images**: Periodically rebuild Docker images to include the latest security patches and updates.
- **Monitor Backups**: Ensure that automated backups are running as expected and verify the integrity of the backup files.
- **SSL Certificates**: If using SSL, monitor the expiration date of your certificates and renew them as needed.

### Conclusion

This guide provides a comprehensive approach to backing up, Dockerizing, and migrating your Koken CMS. By following the steps outlined and testing thoroughly, you can minimize the risk of issues and ensure a smooth transition to your new server environment.
