# Deploying the Cloud86 Training Portal to Plesk

This guide explains how to deploy the Symfony-based Training Portal (featuring Package Pro & DNS Detective) to a Plesk web hosting server.

## 1. Prerequisites
- A Plesk hosting account with SSH access available.
- PHP 8.2 or higher enabled for the domain.
- Composer installed on the server (usually pre-installed on Plesk).
- A domain or subdomain configured (e.g., `training.cloud86.io`).

## 2. Preparing the Plesk Domain
1. Log into your Plesk Control Panel.
2. Select your domain and go to **Hosting & DNS > Hosting Settings**.
3. **Important**: Change the **Document Root** from `httpdocs` to `httpdocs/public`. Symfony serves all its secure traffic out of the `public/` directory to prevent unauthorized access to configuration files.
4. Ensure **PHP Support** is enabled and set to 8.2+ (FPM application served by nginx/Apache).

## 3. Uploading the Code
You can upload the code via Git or SFTP:

### Option A: Using Plesk Git (Recommended)
1. In Plesk, click on the **Git** extension for your domain.
2. Connect your repository and configure it to deploy to the `httpdocs` folder automatically.

### Option B: Using SFTP / File Manager
1. Zip your local project directory (excluding the `vendor/`, `var/`, and `.git/` folders).
2. Upload the zip to the `httpdocs` folder via the Plesk File Manager.
3. Extract the contents.

## 4. Environment Configuration
1. In the `httpdocs` folder, duplicate the `.env` file and rename it to `.env.local`.
2. Edit `.env.local` to configure the production environment:
   ```env
   APP_ENV=prod
   APP_DEBUG=0
   ```
3. Since we are using an SQLite database, ensure that your `DATABASE_URL` is pointing to the correct absolute path inside the `var/` folder, and that the web server has read/write permissions for that directory.
   ```env
   DATABASE_URL="sqlite:///%kernel.project_dir%/var/data.db"
   ```

## 5. Installing Dependencies
It is best to install dependencies directly on the server via SSH.
1. Open the **SSH Terminal** in Plesk (or connect via your terminal app).
2. Navigate to the project root:
   ```bash
   cd /var/www/vhosts/yourdomain.com/httpdocs
   ```
3. Run Composer to install production dependencies:
   ```bash
   composer install --no-dev --optimize-autoloader
   ```

## 6. Database Migrations and Setup
Since the database is SQLite, you don't need to configure a separate MySQL database in Plesk. We just need to initialize it:
1. Still in the SSH terminal, run the Doctrine migrations to build the schema:
   ```bash
   php bin/console doctrine:migrations:migrate --no-interaction
   ```
2. Clear the Symfony cache for the production environment:
   ```bash
   php bin/console cache:clear --env=prod
   ```

## 7. Populating the Apps & Course Content
To ensure the applications (Package Pro & DNS Detective) appear in your portal, you need to populate the database.
1. Run the custom seed command to insert the cases, questions, and app definitions:
   ```bash
   php bin/seed_package_pro.php
   ```
*(Note: Ensure that the script has the exact same paths available on production.)*

## 8. Promoting an Administrator
To access the EasyAdmin control panel at `/admin`, you will need an account with `ROLE_ADMIN`.
1. Register an account normally via the `/register` page on your active domain.
2. Via Plesk SSH, manually promote your user using the Symfony Doctrine command:
   ```bash
   php bin/console dbal:run-sql "UPDATE user SET roles = '[\"ROLE_ADMIN\"]' WHERE email = 'your.email@cloud86.io'"
   ```

## 9. Final Permissions Check
SQLite requires the database file *and* the folder containing it to be writable by the web server (usually `www-data` or the Plesk system user).
1. In Plesk File Manager, navigate to the `var/` folder.
2. Ensure the permissions for `var/` and `var/data.db` allow the web server to write to them (typically `775` or `766`).
3. If using SSH:
   ```bash
   chmod -R 775 var/
   chown -R username:psacln var/
   ```

Your Training Portal is now live and ready to accept trainee registrations!
