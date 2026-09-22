# Cla'ads Property Listing Platform

A portfolio-ready full-stack property marketplace built with Core PHP, MySQL, JavaScript, Bootstrap, HTML, and CSS. Sellers submit property listings, administrators review them, and buyers verify their phone before viewing the owner's contact details.

## Main features

- Search and filter published properties by keyword, sale/rent, property type, and city
- Seller submission form with server-side validation and up to six property photos
- Private seller management link with listing status and “mark as sold” action
- Administrator authentication, approval/rejection, featuring, and audit logs
- OTP-protected owner contact details with expiry, attempt limits, and rate limiting
- Automatic 30-day listing expiry through a CLI cron job
- Authenticated Google Sheets import API with duplicate prevention and import logs
- Prepared SQL statements, CSRF protection, escaped output, MIME-checked uploads, and password hashing
- Docker-based local setup and responsive Bootstrap interface

## Technology

- PHP 8.3 with PDO
- MySQL 8.4
- Bootstrap 5.3 and vanilla JavaScript
- Apache and Docker Compose
- Google Apps Script integration for Google Sheets

## Quick start with Docker

Requirements: Docker Desktop or Docker Engine with Compose.

1. Start the application and database:

   ```bash
   docker compose up --build -d
   ```

2. Create an administrator. Use a password with at least 12 characters:

   ```bash
   docker compose exec web php tools/create-admin.php "Joy Patel" joy@example.com "change-this-password"
   ```

3. Add a sample published listing:

   ```bash
   docker compose exec web php tools/seed-demo.php
   ```

4. Open the application:

   - Public marketplace: `http://localhost:8080`
   - Administrator sign-in: `http://localhost:8080/admin/login.php`

The MySQL service is exposed on local port `3307` for optional database tools. Application data persists in the `claads_mysql` Docker volume.

## Local setup without Docker

Requirements: PHP 8.2+, the `pdo_mysql` and `fileinfo` extensions, MySQL 8+, and a web server whose document root points to `public/`.

1. Create the database with `database/schema.sql`.
2. Copy `config/config.example.php` to `config/config.php`.
3. Update the database connection, `base_url`, `app_key`, and `import_api_key`.
4. Give the web-server user write access to `public/uploads/` and `storage/logs/`.
5. Create an admin with `php tools/create-admin.php ...`.

Never commit `config/config.php`, production credentials, uploaded property photos, or OTP logs. These paths are excluded by `.gitignore`.

## OTP behaviour

The default development driver writes OTP codes to `storage/logs/otp.log` and also displays the current code in a flash message. This makes the complete workflow easy to demonstrate locally without paying for an SMS provider.

Before production deployment, replace the log block in `app/OtpService.php` with a real provider such as Twilio Verify or AWS SNS, set `APP_ENV=production`, use a strong random `APP_KEY`, serve the application over HTTPS, and stop exposing OTP values in responses or logs.

Verified contact access lasts 30 minutes by default and is limited to the individual property.

## Google Sheets import

The endpoint is `POST /api/import-property.php`. It accepts JSON or form data and requires this header:

```text
X-API-Key: your-import-api-key
```

Required fields:

```text
external_id, final_submit=yes, listing_for, property_type, title,
city, area, price, owner_name, owner_phone
```

Optional fields:

```text
sub_type, description, full_address, map_url, area_size, area_unit,
bedrooms, bathrooms, furnished, amenities, owner_email, expires_at
```

Example request:

```bash
curl -X POST http://localhost:8080/api/import-property.php \
  -H "Content-Type: application/json" \
  -H "X-API-Key: local-import-key-change-before-deploying" \
  -d '{
    "external_id": "sheet-row-1001",
    "final_submit": "yes",
    "listing_for": "rent",
    "property_type": "apartment",
    "title": "One-bedroom apartment near transit",
    "city": "Brantford",
    "area": "Downtown",
    "price": 1850,
    "owner_name": "Property Owner",
    "owner_phone": "5195550199"
  }'
```

Every imported property starts in `pending` status. An administrator must publish it. Reusing an `external_id` returns HTTP 409 and does not create a duplicate.

To connect a spreadsheet:

1. Add these header columns: `external_id`, `final_submit`, the required property fields above, and `import_status`.
2. Open Extensions → Apps Script and paste `integrations/google-sheets.gs`.
3. In Apps Script Project Settings, add `CLAADS_API_URL` with the complete endpoint URL and `CLAADS_API_KEY` with the configured key.
4. Run `importApprovedRows()` manually, or run `createHourlyImportTrigger()` once to schedule hourly imports.
5. Set `final_submit` to `yes` only when a row is ready. The script writes the API result to `import_status`.

## Automatic expiry

Run the expiry command daily:

```bash
docker compose exec -T web php cron/expire-listings.php
```

Example host cron entry:

```cron
15 2 * * * cd /absolute/path/to/claads-platform && docker compose exec -T web php cron/expire-listings.php
```

## Project structure

```text
app/            Database, repository, OTP service, and shared helpers
config/         Environment-aware configuration
cron/           Automatic listing-expiry command
database/       MySQL schema
integrations/   Google Apps Script importer
public/         Apache document root, web pages, API, assets, and uploads
public/admin/   Moderation dashboard
public/api/     Authenticated import endpoint
tools/          Admin creation and demo-data commands
```

## Verification checklist

1. Submit a seller listing and save its private management URL.
2. Sign in as admin and publish the pending property.
3. Find the property through public search.
4. Request an OTP on the property page and use the development code shown.
5. Confirm that contact information becomes visible only for that property.
6. Mark the listing as sold and confirm it disappears from public results.
7. Import a Google Sheets row and confirm a repeated `external_id` is rejected.

## Production hardening

This repository is a complete portfolio MVP, not a managed production service. Before public deployment, add a real SMS provider, centralized logs, backups, monitoring, administrator login throttling, email verification, CAPTCHA or stronger abuse controls, image malware scanning and resizing, stricter Content Security Policy headers, and automated integration tests. Store all secrets in the deployment platform rather than source control.

## Resume-safe project description

Use only claims you have personally implemented and can explain in an interview. After running, testing, and understanding this project, a truthful description is:

> Built a responsive property listing platform using Core PHP, MySQL, JavaScript, and Bootstrap, featuring seller submissions, admin moderation, secure image uploads, searchable listings, OTP-protected owner contact details, automatic expiry, and an authenticated Google Sheets import API with duplicate prevention.
