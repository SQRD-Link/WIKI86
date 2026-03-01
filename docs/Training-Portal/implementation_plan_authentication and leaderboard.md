# Implementation Plan: User Tracking & Leaderboard

## Goal Description
The objective is to allow trainees to log into the Training Portal, track their progress in the connected apps (DNS Detective & Package Pro), and display a global leaderboard for healthy competition.

Currently, progress in DNS Detective and Package Pro is saved locally in the browser's `localStorage` (e.g., `package_pro_progress`). We need to transition this to a persistent backend workflow using the Symfony database and an explicit user authentication system.

## Proposed Changes

### 1. User Authentication System
We will implement a standard Symfony authentication flow.
- Ensure the [User](file:///Users/link/Documents/projects/training-portal/src/Entity/User.php#10-109) entity implements `UserInterface` and `PasswordAuthenticatedUserInterface`.
- Configure [config/packages/security.yaml](file:///Users/link/Documents/projects/training-portal/config/packages/security.yaml) with a JSON login route (for the apps to hit) and an interactive web login route (for a future frontend portal).
- Add controllers for Registration and Login.

### 2. API Endpoints for Progress Tracking
Since API Platform is already installed and `#[ApiResource]` is present on [App](file:///Users/link/Documents/projects/training-portal/src/Entity/App.php#6-21) and [UserProgress](file:///Users/link/Documents/projects/training-portal/src/Entity/UserProgress.php#6-26), we will leverage it instead of writing custom controllers from scratch.
- **[MODIFY]** [src/Entity/UserProgress.php](file:///Users/link/Documents/projects/training-portal/src/Entity/UserProgress.php) and [src/Entity/CompletedStep.php](file:///Users/link/Documents/projects/training-portal/src/Entity/CompletedStep.php): Ensure they are mapped correctly so the frontend can `POST` and `GET` progress via the standard `/api/user_progresses` routes.
- **[MODIFY]** Configure API Platform to automatically link the currently logged-in user to the progress records they create (using a custom DataPersister or StateProcessor).
- **[MODIFY]** [public/package-pro/app.js](file:///Users/link/Documents/projects/training-portal/public/package-pro/app.js) and [public/dns-detective/app.js](file:///Users/link/Documents/projects/training-portal/public/dns-detective/app.js): Remove `localStorage` reliance. Update the [saveProgress()](file:///Users/link/Documents/projects/training-portal/public/package-pro/app.js#12-28) and [loadProgress()](file:///Users/link/Documents/projects/training-portal/public/package-pro/app.js#29-38) functions to make `fetch()` calls to the API Platform endpoints (`/api/user_progresses`).

### 3. Leaderboard Implementation
We will create a query to aggregate XP across all apps and display it.
- **[NEW]** `src/Controller/Api/LeaderboardController.php`:
  - `GET /api/leaderboard`: Returns a JSON array of the top 10/20 users ranked by total XP across all connected [App](file:///Users/link/Documents/projects/training-portal/src/Entity/App.php#6-21) entities.
- **[MODIFY]** Add a "Leaderboard" modal or section to the frontend applications (or a central portal homepage) that fetches and displays this data.

## Verification Plan

### Automated/Manual Verification
1. Create a dummy test user via the Symfony console or API.
2. Log into the DNS Detective frontend using those credentials.
3. Complete Case #1. Verify via the API and EasyAdmin Dashboard that a [UserProgress](file:///Users/link/Documents/projects/training-portal/src/Entity/UserProgress.php#6-26) row was created and updated correctly in the database.
4. Refresh the page to verify that [loadProgress()](file:///Users/link/Documents/projects/training-portal/public/package-pro/app.js#29-38) successfully fetches the state from the backend rather than local storage.
5. Hit the `/api/leaderboard` endpoint to confirm the user's XP is correctly aggregated and ranked.
