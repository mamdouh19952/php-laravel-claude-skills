---
name: shared-hosting-deploy
description: 'Deploy a Laravel app or a native PHP + MySQL site to shared hosting (cPanel, Hostinger hPanel, or similar) — on the main domain, a subdomain, or a subfolder like example.com/app — without exposing .env/config/vendor, and fix the usual traps (Composer platform/PHP version mismatch, permissions after a Windows zip, 405/404 on a symlinked public folder, stale route/config cache, storage:link blocked, no Supervisor for queues, 500 after upload). Use whenever deploying or updating a PHP site on shared hosting, choosing subdomain vs subfolder, setting up cron/queues there, or debugging a live 500/405/"Composer platform" error. Trigger words: deploy, cPanel, hPanel, Hostinger, shared hosting, public_html, رفع المشروع على السيرفر, استضافة.'
---

# Deploying PHP to shared hosting

Shared hosting (cPanel, Hostinger hPanel…) gives you `public_html` as the web root, SSH or a web
Terminal on most plans, MySQL, and cron — but usually no root, no Supervisor, and some PHP functions
disabled. Everything below works within those limits. Replace `example.com`, `<user>`, `<project>`
with the real values.

## 1. Never expose the project root

`public_html` (and every folder inside it) is reachable by URL. `.env`, `config.php`, `app/`,
`vendor/`, `storage/`, `composer.json`, SQL dumps must **not** be inside it.

**Laravel:** put the project next to `public_html`, and expose only `public/`:

```bash
# project lives outside the web root
~/<project>-app/            # .env, app/, vendor/, storage/ ...

# main domain: replace public_html with a link to public/
mv ~/public_html ~/public_html_old
ln -s ~/<project>-app/public ~/public_html

# or a subfolder URL: example.com/<folder>
ln -s ~/<project>-app/public ~/public_html/<folder>
```

On hPanel the domain root is usually `~/domains/example.com/public_html` — same idea.
If the panel lets you set a domain/subdomain **document root**, point it at `~/<project>-app/public`
instead of a symlink.

**Native PHP:** the site itself must live in the web root, so keep secrets out of it:
`config.php` (DB credentials) above `public_html` when possible
(`require __DIR__ . '/../../config.php'`), or keep it inside with the `.htaccess` deny rules and
`inc/` / `db/` folders denied (`Require all denied`), and no PHP execution in `uploads/`.

Never accept "nobody will look" for an exposed `.env` — if it's reachable by URL, it's leaked.
Check: open `https://example.com/.env` and `https://example.com/config.php` → must be 403/404.

## 2. Subfolder vs subdomain

- **Subfolder** (`example.com/<folder>`): just the symlink above; no DNS. In Laravel's `.env`:
  ```
  APP_URL=https://example.com/<folder>
  ASSET_URL=https://example.com/<folder>
  SESSION_PATH=/<folder>
  ```
- **Subdomain** (`app.example.com`): create it in the panel, set its document root to the project's
  `public/` (Laravel) or site folder (native), enable SSL for it. `APP_URL=https://app.example.com`.

## 3. Match Composer to the server's PHP — before uploading (Laravel)

`"php": "^8.2"` is wide open: `composer update` on your machine picks package versions for *your*
PHP (say 8.5) and locks them; the server's older PHP (say 8.3) then refuses to install them.

```bash
php -v          # on the SERVER (SSH / Terminal) — and check the site's PHP in MultiPHP / PHP settings
```

Pin it in `composer.json`, then re-resolve locally and run the tests:

```json
"config": {
    "platform": { "php": "8.3.0" }
}
```

```bash
composer update --with-all-dependencies
php artisan test
```

Upload `composer.json` **and** `composer.lock`. Note: the PHP version in SSH can differ from the
website's PHP version — set both to the same version in the panel.

Also enable the PHP extensions the app needs in the panel's PHP settings — at least `pdo_mysql`,
`mbstring`, `fileinfo`, `openssl`, and **`gd` or `imagick` if the app resizes/converts images**.

## 4. Upload

Prefer `git clone`/`git pull` over SSH. For a **private repo**: `ssh-keygen` on the server, add the
public key as a read-only **Deploy key** in the repo settings, clone with the SSH URL.

If you must upload a zip: build a clean one (exclude `node_modules`, `.git`, `vendor`, `.env*`,
`storage/logs/*`, tests), build assets locally first (`npm run build`), extract on the server.

Some hosts ship Composer 1 and 2 side by side — use `composer2` there.

## 5. Permissions after a Windows zip

Extracting a Windows zip can leave folders without execute permission (`Permission denied` from
Composer, or a 500):

```bash
find ~/<project>-app -type d -not -path "*/vendor/*" -exec chmod 755 {} \;
find ~/<project>-app -type f -not -path "*/vendor/*" -exec chmod 644 {} \;
chmod -R 775 ~/<project>-app/storage ~/<project>-app/bootstrap/cache   # Laravel
chmod 775 ~/public_html/uploads                                        # native PHP uploads
```

## 6. First install (Laravel)

```bash
cd ~/<project>-app
composer install --no-dev --optimize-autoloader      # or composer2
cp .env.example .env && nano .env                    # production values, below
php artisan key:generate                             # only on the first install — never change it later
php artisan migrate --force
php artisan storage:link
php artisan optimize
```

If `storage:link` is blocked (`symlink()` disabled), create the link by hand:

```bash
ln -s ~/<project>-app/storage/app/public ~/<project>-app/public/storage
```

Production `.env` essentials:

```
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com
LOG_CHANNEL=daily
LOG_LEVEL=warning
DB_HOST=localhost
DB_DATABASE=<user>_<db>
DB_USERNAME=<user>_<dbuser>
DB_PASSWORD=<long random password>
QUEUE_CONNECTION=database
```

## 7. First install (native PHP)

1. Create the database + user in the panel (MySQL Databases); give the user all privileges on it.
2. Import `db/schema.sql` with phpMyAdmin (Import tab).
3. Upload the site into `public_html` (or `public_html/<folder>`).
4. Create `config.php` from `config.example.php` with the real credentials and `'debug' => false`.
5. Set the PHP version (MultiPHP Manager / PHP settings) and enable `pdo_mysql`, `mbstring`,
   `fileinfo`, `gd`; make sure `display_errors` is Off.
6. Make `uploads/` writable (775) and confirm PHP can't run there (its `.htaccess`).

## 8. `.htaccess` when Laravel's `public/` is reached through a symlink (405 / 404 on the root URL)

On some hosts (LiteSpeed especially) the bare folder URL doesn't reach `index.php` through a
symlinked directory and returns `405 Method Not Allowed` before Laravel runs. Harden
`public/.htaccess`:

```apache
# Redirect Trailing Slashes If Not A Folder...
RewriteCond %{REQUEST_FILENAME} !-d
RewriteCond %{REQUEST_URI} (.+)/$
RewriteRule ^ %1 [L,R=301]

DirectoryIndex index.php

# Send Requests To Front Controller...
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule ^ index.php [L]
```

(vs Laravel's default: adds `DirectoryIndex index.php`, and drops the `!-d` condition from the last
rule so every non-file request goes to `index.php`.)

## 9. After any update or dependency change — clear caches first

A `405 … Supported methods: HEAD` on a route that is clearly `Route::get()`, or `env()` returning
null, is usually a stale cache:

```bash
php artisan optimize:clear      # config, routes, views, cache
php artisan optimize            # rebuild once everything works
```

Every later deploy:

```bash
cd ~/<project>-app
git pull origin main
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan optimize
php artisan queue:restart       # if a queue is used
```

Any edit to `.env` on the server → `php artisan optimize` again (cached config ignores `.env`).
Use `env()` only inside `config/*.php`; code reads `config()`.

## 10. Scheduler and queues without Supervisor

Add **one** cron job in the panel (Cron Jobs), every minute — find the PHP path with `which php`:

```
* * * * * /usr/bin/php /home/<user>/<project>-app/artisan schedule:run >> /dev/null 2>&1
```

No always-on worker on shared hosting, so let the scheduler run a short-lived one that drains the
queue and exits:

```php
// routes/console.php
Schedule::command('queue:work --stop-when-empty --max-time=55')
    ->everyMinute()
    ->withoutOverlapping();
```

If the app has no real async work, keep `QUEUE_CONNECTION=sync` and skip this.

## 11. A 500 after deploying — check in this order

1. The log: `tail -n 50 storage/logs/laravel.log` (native PHP: the host's `error_log` file / panel Errors).
2. Permissions on `storage` and `bootstrap/cache` (or `uploads/`).
3. Missing `APP_KEY` → `php artisan key:generate` (first install only).
4. `.env` edited after `optimize` → `php artisan optimize:clear` then `optimize`.
5. Wrong PHP version for the site, or a missing extension (step 3).
6. Still stuck: set `APP_DEBUG=true` (native: `'debug' => true`) **briefly** to see the real error,
   then set it back to `false` and `php artisan config:clear` the moment you know. Never leave a live
   site in debug mode.

## 12. Launch checklist

| ✓ | Check | How |
|---|---|---|
| ☐ | `APP_ENV=production`, `APP_DEBUG=false` (native: `debug => false`) | Open a wrong URL → a plain error page, no details |
| ☐ | `.env` / `config.php` / `vendor` / `storage` not reachable | Request them by URL → 403/404 |
| ☐ | HTTPS on, `APP_URL` uses https | Images and assets load over https |
| ☐ | Tests pass locally (and CI if present) | `php artisan test` |
| ☐ | Caches built | `php artisan optimize` |
| ☐ | `storage` / `bootstrap/cache` / `uploads` writable | Upload an image |
| ☐ | Images optimized on upload (if the app shows user images) | Upload a big photo → a small webp is stored |
| ☐ | Cron + queue running (if used) | `php artisan schedule:list`; trigger an email and see it arrive |
| ☐ | Rate limiting on login | 6 wrong logins in a row → 429 |
| ☐ | CORS limited to the frontend domain (APIs) | `config/cors.php` — no `*` |
| ☐ | No secrets in git history | `git log -p \| grep -iE "password\|secret\|api_key"` |
| ☐ | No known vulnerable packages | `composer audit` |
| ☐ | Daily database backup — and one restore tested | Panel backups or `spatie/laravel-backup` |
| ☐ | Uptime monitor | Laravel's `/up` route in UptimeRobot / Better Stack |
| ☐ | Walk the real user flow on the live URL | Login, create, edit, delete — no console/network errors |

## Official docs

- Laravel deployment: `https://laravel.com/docs/{version}/deployment` (version from `composer.json`).
- Laravel scheduling / queues: `/scheduling`, `/queues` on the same site.
- Your host's own knowledge base for panel specifics (PHP version, cron, SSH, document roots).
