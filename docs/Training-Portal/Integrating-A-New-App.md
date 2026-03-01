# Integrating a New Application (Package Pro) Tutorial

This tutorial explains step-by-step how we integrated "Package Pro" into the Training Portal. The goal is for you to understand the *why* and *how* behind each change, from the frontend static files all the way to the Symfony backend database and EasyAdmin dashboard.

Whether you replicate this for another app or just want to understand the codebase better, these chapters will guide you through the process we used.

---

## Chapter 1: The Frontend (Static Files)

**What we did:** We copied the `package-pro` folder into the `public/` directory of the Symfony project.

**Why:** The Training Portal is designed with a decoupled architecture. The "games" (DNS Detective, Package Pro) are standalone static applications written in raw HTML, CSS, and JS. Symfony does not use Twig to render their inner logic. By placing the folder in `public/package-pro`, the web server (or the Symfony local server) serves those static files directly at `http://localhost:8000/package-pro/`. 

This keeps the frontend completely isolated. Symfony's main job is simply tracking *progress* and serving as an admin interface to manage the data.

---

## Chapter 2: The Database "App" Entity

To make Package Pro show up in the EasyAdmin dashboard (the portal backend), we needed to tell the database that this app exists. The blueprint for database tables in Symfony is an **Entity**.

### Adding Missing Fields
**What we did:** We modified `src/Entity/App.php`. We noticed that when we tried to seed the database, properties like `description`, `icon`, `colorTheme`, and `isActive` were missing, causing a fatal PHP error.

**Why:** In Doctrine (Symfony's database ORM tool), PHP class properties mapped with `#[ORM\Column]` attributes define the actual table columns.
We added:
```php
#[ORM\Column(type: 'text', nullable: true)]
public ?string $description = null;

#[ORM\Column(length: 50, nullable: true)]
public ?string $icon = null;

#[ORM\Column(length: 50, nullable: true)]
public ?string $colorTheme = null;

#[ORM\Column(type: 'boolean', options: ['default' => true])]
public bool $isActive = true;
```

*Note on `isActive`:* We initially forgot the `options: ['default' => true]`. Because the database *already had data* (the DNS Detective row), adding a `NOT NULL` boolean column (`bool $isActive` without a `?`) made SQLite/Database complain: "Cannot add a NOT NULL column without a default value." We fixed that by providing the default value directly in the ORM attribute.

---

## Chapter 3: Running Migrations

**What we did:** We ran `php bin/console make:migration` and then `php bin/console doctrine:migrations:migrate`.

**Why:** Editing an Entity file in PHP does not automatically change the database.
1. `make:migration` compares your PHP files to your current database schema and generates a raw SQL file (`VersionXXXX.php` in the `migrations/` folder) containing the `ALTER TABLE` commands needed to bridge the gap.
2. `doctrine:migrations:migrate` safely executes that SQL file. This ensures the database stays perfectly synced with your code.

---

## Chapter 4: Seeding The Data (The Tricky Part!)

To create the initial data, we wrote a PHP script: `bin/seed_package_pro.php`.

### Step 4.1: The `App` Table
**What we did:** We created a new instance of the `App` entity and assigned it the values for Package Pro.
```php
$app = new App();
$app->name = 'Package Pro';
$app->slug = 'package-pro';
// ... set description, icon, etc.
$em->persist($app);
```
**Why:** `$em` is the `EntityManager`. Calling `persist()` tells Doctrine "start tracking this object". Calling `flush()` actually writes it to the database with an `INSERT` statement.

### Step 4.2: Realizing We Forgot the Cases!
Initially, we stopped there. Package Pro showed up on the EasyAdmin dashboard... but it was completely empty inside. It had no "Course Cases" or "Quiz Questions". We hadn't seeded them!

### Step 4.3: Extracting Data from Javascript
**What we did:** The `app.js` file for Package Pro contained a massive `CASES` Javascript object storing all the scenario text, questions, and answers. Typing them out by hand in PHP would be terrible.
Instead, we:
1. Wrote a quick Node.js script to extract the `CASES` block from `app.js` and save it as a JSON file (`/tmp/package_pro_cases.json`).
2. Updated our PHP seed script to `file_get_contents()` and `json_decode()` that JSON back into a PHP array.
3. Created `CourseCase`, `Step`, and `Question` entities in a loop for every case we found.

```php
foreach ($casesData as $data) {
    // 1. Create the Course Case
    $case = new CourseCase();
    $case->app = $app; // Link it back to the main App!
    $case->title = $data['title'];
    $em->persist($case);

    // 2. Loop through steps in the case
    foreach ($data['steps'] as $stepData) {
        $step = new Step();
        $step->courseCase = $case; // Link it back to the Case
        // ... set step properties
        $em->persist($step);

        // 3. Loop through quiz questions in the step
        foreach ($stepData['questions'] as $qData) {
            $q = new Question();
            $q->step = $step; // Link it back to the Step
            // ... set question properties
            $em->persist($q);
        }
    }
}
$em->flush(); // Commit all 150+ rows to the database at once!
```
**Why:** A relational database requires these connections. An `App` One-to-Many `CourseCases`. A `CourseCase` One-to-Many `Steps`. And a `Step` One-to-Many `Questions`. Setting these properties effectively creates the Foreign Key relationships.

---

## Chapter 5: Fixing the EasyAdmin Dashboard Error

**What we did:** Early on, when you tried to load `/admin`, EasyAdmin threw a fatal runtime error: `"Call to undefined method MenuItem::linkToCrud()"`. We fixed this in `DashboardController.php`.

**Why:** The code you had earlier was for EasyAdmin version 3.x. However, the project's `composer.json` specified `"easycorp/easyadmin-bundle": "*"`, which installed the newest Version 4.x. 

In EasyAdmin 4, `MenuItem::linkToCrud()` was deleted. Replacing it was easy:
```diff
- yield MenuItem::linkToCrud('Apps', 'fa fa-graduation-cap', App::class);
+ yield MenuItem::linkTo(AppCrudController::class, 'Apps', 'fa fa-graduation-cap');
```
You now pass the fully qualified class name of the *Controller* rather than the *Entity*, using the newer `linkTo()` method.

---

## Summary

You now have a complete, working knowledge of the data pipeline for your portal!
1. **Frontend:** Static HTML/JS in `public/{app-name}`
2. **Backend Structure:** `src/Entity/{Name}.php` models the tables.
3. **Database Changes:** `make:migration` -> `migrate`
4. **Data Population:** Custom `bin/seed_{app}.php` scripts to parse text/JSON and `persist()` objects.
5. **Dashboard UI:** EasyAdmin `...CrudController.php` files automatically render grids based on the database data!
