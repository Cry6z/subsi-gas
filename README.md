# Subsi-Gas

Stack: Laravel 12 + Livewire + Flux UI + TailwindCSS + MySQL.

## Requirements

- PHP 8.2+
- Composer
- Node.js + npm
- MySQL

## Setup

1. Install dependencies

```bash
composer install
npm install
```

2. Create `.env`

Copy your `.env` manually (this repo may not ship a `.env.example`). Minimal keys:

```env
APP_NAME="subsi_gas"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=subsi_gas
DB_USERNAME=root
DB_PASSWORD=

# Optional (Leaflet can use OpenStreetMap tiles without a token)
MAPBOX_TOKEN=
# Optional alternative
GOOGLE_MAPS_API_KEY=
```

Generate key:

```bash
php artisan key:generate
```

3. Migrate + seed

```bash
php artisan migrate --seed
```

4. Storage (photo uploads)

```bash
php artisan storage:link
```

5. Run

```bash
npm run dev
php artisan serve
```

## Default Seeded Accounts

All seeded users use password: `password`

- Admin
  - email: `admin@example.com` (or `ADMIN_EMAIL`)
- Distributors
  - created by seeder (3 accounts)

## Routing / Role Redirect

- After login, users go to `/dashboard` which redirects by role:
  - `admin` -> `/admin/dashboard`
  - `distributor` -> `/distributor/dashboard`

Role middleware:

- `role:admin`
- `role:distributor`

## Maps

This project uses **Leaflet**.

- If `MAPBOX_TOKEN` is provided, tiles will use Mapbox.
- If empty, it falls back to OpenStreetMap tiles.

### Nearby Search (Haversine)

Production (MySQL) uses SQL Haversine:

```sql
SELECT id, name, address, latitude, longitude, stock,
  (6371 * acos(
    cos(radians(:lat)) * cos(radians(latitude))
    * cos(radians(longitude) - radians(:lng))
    + sin(radians(:lat)) * sin(radians(latitude))
  )) AS distance
FROM locations
HAVING distance <= :radius
ORDER BY distance ASC;
```

Tests use SQLite in-memory; in that case the app falls back to computing distances in PHP.

## Admin creates distributor (invite flow)

- Distributors do not self-register.
- Admin creates distributor accounts via Admin UI.
- The system does not email plaintext passwords. It sends a password reset link so the distributor can set their own password.

Mail must be configured in `.env` for invites to be delivered:

```env
MAIL_MAILER=smtp
MAIL_HOST=
MAIL_PORT=
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=no-reply@example.com
MAIL_FROM_NAME="${APP_NAME}"
```

## Notes

- Two-factor authentication is disabled.
- Email verification is disabled.
