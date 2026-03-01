# Training Portal Technical Setup & Configuration

## Overview
The Training Portal is a Symfony 7/8 application serving as a unified platform to host various training applications like **DNS Detective** and **Package Pro**. It provides an administrative backend built with EasyAdmin for managing courses, interactive cases, users, and progress tracking.

## Architecture
- **Backend Framework:** Symfony (latest Stable 7.x/8.x)
- **Language Requirements:** PHP 8.4+
- **Database:** PostgreSQL (dev handled via Docker `compose.yaml`)
- **ORM System:** Doctrine ORM
- **Admin Interface:** EasyCorp/EasyAdminBundle 4.x
- **Frontend Architecture:** Static HTML/CSS/JS applications mounted within the `public/` directory.

## Core Applications Hosted
1. **DNS Detective**
   - **Path:** `public/dns-detective/`
   - **Type:** Single Page App (JS/HTML/CSS)
   - **Seeding:** `bin/seed_dns_detective.php`
2. **Package Pro**
   - **Path:** `public/package-pro/`
   - **Type:** Single Page App (JS/HTML/CSS)
   - **Seeding:** `bin/seed_package_pro.php`

Both applications are registered in the main relational database as `App` entities and can be modified via the EasyAdmin interface.

---

## Local Development Setup

To onboard developers to the project locally, run the following steps:

### 1. Start the Database Container
A standard PostgreSQL Docker image is provided.
```bash
docker compose up -d
```

### 2. Install Dependencies
```bash
composer install
```

### 3. Environment Variables
Ensure your `.env` or `.env.local` contains the correct `DATABASE_URL`. The default Docker setup expects:
```env
DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:5432/app?serverVersion=16&charset=utf8"
```

### 4. Apply Database Migrations
Create the necessary tables (Apps, CourseCases, Steps, Questions, Users, Progress):
```bash
php bin/console doctrine:migrations:migrate --no-interaction
```

### 5. Seed the Database
Populate the initial syllabus using the custom seed scripts:
```bash
php bin/seed_dns_detective.php
php bin/seed_package_pro.php
```

### 6. Start the Server
```bash
symfony server:start -d
# Admin UI accessible at: http://localhost:8000/admin
```

---

## Database Schema Highlights

The relational schema heavily leverages Doctrine attributes.
- `App`: Top-level entity (name, slug, description, icon, theme, isActive).
- `CourseCase`: Dependent on `App` (title, xpReward, sortOrder).
- `Step` & `Question`: Content building blocks associated directly to a `CourseCase`.
- `User`, `UserProgress`, `CompletedStep`: Tracking progress across multiple apps and cases.

**Data Iteration Note:** 
If new application entities require new properties, follow standard Doctrine migration flows: update the Entity PHP class properties (with e.g. `#[ORM\Column]`), then run `make:migration`, followed by `doctrine:migrations:migrate`.

## Key Technical Decisions
- **Migration from EasyAdmin 3.x to 4.x Standards:** The `DashboardController.php` menu items leverage `MenuItem::linkTo(Controller::class)` instead of the deprecated `MenuItem::linkToCrud()`.
- **Decoupled Frontend:** The frontend "games" (DNS Detective & Package Pro) do not use Twig routing. They are statically served standalone assets within `/public`. The Symfony backend strictly acts as their administrative layer.
