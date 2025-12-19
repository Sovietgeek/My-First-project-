CarbonCalc — Personal Carbon Footprint Platform

## Project overview
CarbonCalc helps individuals and organizations measure, monitor, and reduce their carbon footprint.
It provides personalized carbon calculations from a lifestyle survey, ongoing tracking (daily/weekly/monthly), goal-based reduction planning, gamification (badges, leaderboards, eco-challenges), and a marketplace for carbon offset actions (tree planting, carbon credits).

Key modules:
- Module A: User Management & Authentication
- Module B: Carbon Calculator & Surveys
- Module C: Goals & Gamification
- Module D: Eco Marketplace & Transactions

NOTE: The original statement lists Spring Boot (Java) as the backend in the Tech Stack but the Milestone plan mentions FastAPI. I will assume Spring Boot for the backend and provide alternative FastAPI notes where relevant. If you'd rather use FastAPI, tell me and I will adapt scaffolding and examples.

## Chosen Tech Stack (assumption)
- Frontend: React.js + Tailwind CSS (Vite or CRA)
- Backend: Spring Boot (Java, Maven)
- Database: MySQL
- Authentication: JWT (access + refresh tokens)
- Integrations: Carbon Interface API, Open Energy Data, UN Carbon Emissions Datasets

## High-level architecture
- React frontend communicates with Spring Boot REST API over HTTPS.
- Backend exposes authenticated endpoints for surveys, carbon logs, goals, gamification, marketplace and transactions.
- MySQL stores normalized tables (see schema below).
- Background jobs (cron or scheduled tasks) aggregate trends, update leaderboards, and issue alerts.
- Optional message queue (RabbitMQ / Kafka) can be used for marketplace payments and transaction processing.

## 8-Week Milestone Plan (cleaned & aligned)

- Milestone 1: Weeks 1–2 — Auth & Setup
  - Week 1: Initialize Spring Boot + React, create DB schema, implement JWT auth (access + refresh), user registration + profile.
  - Week 2: Design login/register UI and basic dashboard; wire up auth flows.
  - Deliverable: Secure login, profile setup, baseline dashboard.

- Milestone 2: Weeks 3–4 — Carbon Calculation
  - Week 3: Build lifestyle survey form, validate & store responses.
  - Week 4: Implement API-based carbon calculator (Carbon Interface + local formulas) and display history & trends.
  - Deliverable: Functional carbon logs and trends.

- Milestone 3: Weeks 5–6 — Goals & Gamification
  - Week 5: Add goal creation, progress tracking, scheduled reductions.
  - Week 6: Add badges and leaderboard calculation logic, team support.
  - Deliverable: Gamified experience for users.

- Milestone 4: Weeks 7–8 — Marketplace & Alerts
  - Week 7: Add eco marketplace (item listing, purchase flow, simple payment flow/stub), transactions recording.
  - Week 8: Add alerts for high emissions, finalize UI, deploy to cloud (Heroku/Azure/GCP/AWS), finalize docs.
  - Deliverable: Full system with sustainability tools.

## Database schema (DDL)
Below are CREATE TABLE statements for MySQL matching the proposed schema.

-- Users
CREATE TABLE IF NOT EXISTS Users (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  role ENUM('user','admin') NOT NULL DEFAULT 'user',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Surveys
CREATE TABLE IF NOT EXISTS Surveys (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  transport_mode VARCHAR(255),
  diet_type VARCHAR(255),
  energy_usage DECIMAL(12,4),
  frequency JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE
);

-- CarbonLogs
CREATE TABLE IF NOT EXISTS CarbonLogs (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  date DATE NOT NULL,
  transport_emission DECIMAL(12,4) DEFAULT 0,
  food_emission DECIMAL(12,4) DEFAULT 0,
  energy_emission DECIMAL(12,4) DEFAULT 0,
  total_emission DECIMAL(12,4) DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE,
  UNIQUE KEY unique_user_date (user_id, date)
);

-- Goals
CREATE TABLE IF NOT EXISTS Goals (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  goal_title VARCHAR(255) NOT NULL,
  target_emission DECIMAL(12,4) NOT NULL,
  current_emission DECIMAL(12,4) DEFAULT 0,
  status ENUM('active','completed') DEFAULT 'active',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE
);

-- Badges
CREATE TABLE IF NOT EXISTS Badges (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  badge_name VARCHAR(255) NOT NULL,
  description TEXT,
  awarded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE
);

-- Leaderboards
CREATE TABLE IF NOT EXISTS Leaderboards (
  id INT AUTO_INCREMENT PRIMARY KEY,
  team_name VARCHAR(255),
  user_id INT,
  score DECIMAL(12,2) DEFAULT 0,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE SET NULL
);

-- Marketplace
CREATE TABLE IF NOT EXISTS Marketplace (
  id INT AUTO_INCREMENT PRIMARY KEY,
  item_name VARCHAR(255) NOT NULL,
  item_type ENUM('tree_planting','carbon_credit') NOT NULL,
  price DECIMAL(12,2) NOT NULL,
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transactions
CREATE TABLE IF NOT EXISTS Transactions (
  id INT AUTO_INCREMENT PRIMARY KEY,
  user_id INT NOT NULL,
  marketplace_item_id INT NOT NULL,
  amount DECIMAL(12,2) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES Users(id) ON DELETE CASCADE,
  FOREIGN KEY (marketplace_item_id) REFERENCES Marketplace(id) ON DELETE CASCADE
);

## API design (high-level endpoints)
Authentication
- POST /api/auth/register — Register user (name, email, password)
- POST /api/auth/login — Login -> returns { accessToken, refreshToken }
- POST /api/auth/refresh — Accepts refreshToken -> issues new accessToken
- POST /api/auth/logout — Revoke refresh token / logout

Users
- GET /api/users/me — Get current user profile
- PUT /api/users/me — Update profile

Surveys & Carbon
- POST /api/surveys — Submit survey (stores responses)
- GET /api/surveys — List user surveys
- POST /api/carbon/calculate — Trigger calculation for given input or survey
- GET /api/carbon/logs?period=weekly/monthly — Get carbon logs and trends

Goals & Gamification
- POST /api/goals — Create goal
- GET /api/goals — List goals
- POST /api/badges/award — (admin) award badge or automated by rules
- GET /api/leaderboard — Get leaderboard (top N)

Marketplace & Transactions
- GET /api/marketplace — List items
- POST /api/marketplace/purchase — Purchase item (creates transaction)
- GET /api/transactions — List user transactions

Webhooks / Background
- POST /api/webhooks/payment — (optional) payment provider webhook

Security notes (JWT)
- Use short-lived access tokens (e.g., 15 minutes) and long-lived refresh tokens (e.g., 7–30 days).
- Store refresh tokens server-side (DB) with rotation and revocation support.
- Protect endpoints with role checks where necessary (admin-only routes).
- Use HTTPS and set secure cookie flags if storing tokens in cookies.

JWT endpoints and flows
1. User logs in -> server validates credentials -> creates access token (claims: userId, role, exp) and refresh token (random UUID stored in DB with user_id and expiry).
2. Access token used in Authorization: Bearer <token> header for API calls.
3. When access token expires, frontend calls /api/auth/refresh with refresh token (as secure httpOnly cookie or in Authorization header). Server verifies refresh token DB record and issues a new access token (and optionally rotates refresh token).

Error and edge cases
- Handle missing/invalid refresh tokens and implement refresh-token reuse detection.
- Enforce rate-limiting on public endpoints (login, register, calculate) to avoid abuse.
- Validate all survey inputs and apply sanitation for JSON frequency fields.

Directory layout (suggested)

backend/
  ├─ src/main/java/... (Spring Boot app)
  ├─ src/main/resources/application.yml
  ├─ src/test/java/...
  └─ pom.xml

frontend/
  ├─ src/
  ├─ public/
  ├─ package.json
  └─ tailwind.config.js

infra/
  ├─ docker-compose.yml (MySQL, optional queue)
  └─ k8s/ (optional deployment manifests)

## Scaffolding & quick start (Windows PowerShell)

1) Frontend (Vite + React + Tailwind)

Run in PowerShell:

> npm create vite@latest frontend -- --template react
> cd frontend; npm install
> npm install -D tailwindcss postcss autoprefixer; npx tailwindcss init -p

Add Tailwind config and import in src/main.css. Start dev server with:

> npm run dev

2) Backend (Spring Boot + Maven)

Recommended: use https://start.spring.io to prepare the project with dependencies: Spring Web, Spring Data JPA, MySQL Driver, Spring Security, Lombok (optional), Validation.

If you want CLI with curl (or just download zip from start.spring.io):

- Use the generated project or create with Maven archetype. Then in the backend folder run:

> mvn spring-boot:run

Configure DB in src/main/resources/application.yml with your MySQL connection.

3) Docker (local dev)

Create a docker-compose.yml with MySQL and optional adminer. Example (brief):

version: '3.8'
services:
  db:
    image: mysql:8
    environment:
      MYSQL_DATABASE: carboncalc
      MYSQL_ROOT_PASSWORD: rootpassword
    ports: ['3306:3306']

Run in PowerShell:

> docker compose up -d

## Tests & quality gates
- Add unit tests for calculator functions (pure functions) and integration tests for key REST endpoints.
- Add a small test matrix for authentication flows (login, refresh, protected endpoint access).
- Linting: ESLint for frontend, SpotBugs/Checkstyle for Java backend.

## Next steps (what I can do now)
1. Create a separate SQL file with the CREATE TABLE statements in the repo (`infra/schema.sql`).
2. Generate a minimal Spring Boot starter (pom + Application class) and a minimal React + Tailwind skeleton.
3. Implement JWT auth skeleton in backend with refresh token support.

Tell me which of the next steps you'd like me to execute now. If you prefer FastAPI instead of Spring Boot, say so and I'll switch the scaffolding and examples.

---
Generated on: 2025-12-19
# My-First-project- kjhiuhrheghhloh
lnboaerho
