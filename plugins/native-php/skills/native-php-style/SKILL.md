---
name: native-php-style
description: 'Conventions for simple native (plain, no-framework) PHP + MySQL websites — page-per-file with includes, PDO prepared statements, sessions, CSRF, flash messages, safe image uploads, Bootstrap, Arabic/English. Use whenever building or editing a small PHP site without Laravel — a login/register system, a blog, a task manager, a CRUD admin page, a contact form, a small JSON endpoint — or when fixing/reviewing plain PHP code that uses mysqli/PDO, $_POST, $_SESSION, header(''Location''). Trigger words: native PHP, plain PHP, vanilla PHP, PHP and MySQL, mysqli, PDO, no framework, بي اتش بي عادي, نيتف. Not for Laravel projects (use laravel-blade-style / laravel-api-style).'
---

# Native PHP style

For **small, simple** PHP + MySQL sites: a few entities, one developer, shared hosting. Keep it
plain and readable — page-per-file, a handful of helpers, no homemade framework. If the project
grows past that (many entities, a real API, roles and permissions everywhere, a team), recommend
moving to Laravel instead of inventing a router/ORM.

**Match whatever already exists in the project.** Reuse its folders, helper names, and flash
pattern. When the existing code is unsafe (string-built SQL, no CSRF, state changes over GET), fix it
in the code you touch and say so in one line — don't silently copy the unsafe pattern.

## Before writing code

1. Read the project's `CLAUDE.md` (if any); look at the folders, `inc/` helpers, and the SQL schema.
2. Check the PHP version (`php -v`) — this skill assumes PHP 8.1+ and MySQL 5.7+/MariaDB 10.3+.
3. For a multi-step feature, state a short plan first.

## Project structure

One folder per feature; each form has a page that renders it and a `handle_*.php` that processes
it. Shared code lives in `inc/`. Every page is **one folder deep**, so shared links use `../`.

```
project/
├── index.php              → redirects to the main page (e.g. tasks/index.php)
├── handle_lang.php        switches ar/en (allow-list)
├── auth/                  login.php, handle_login.php, register.php, handle_register.php, logout.php
├── tasks/                 index.php, store.php, edit.php, update.php, toggle.php, delete.php
├── inc/                   bootstrap.php, db.php, functions.php, csrf.php, auth.php,
│                          header.php, footer.php, errors.php, success.php, 404.php, lang/ar.php, lang/en.php
│   └── .htaccess          Require all denied
├── api/                   optional: bootstrap.php + one file per resource
├── assets/                css/, js/, images/
├── uploads/               user files (.htaccess: no PHP execution)
├── db/schema.sql          the full schema — kept in sync with every DB change
├── config.example.php     committed; copied to config.php
├── config.php             DB credentials, debug flag — NOT in git
├── .htaccess              no directory listing, deny config.php
└── .gitignore             config.php, uploads/*
```

Every page starts with `require_once __DIR__ . '/../inc/bootstrap.php';` — it loads config,
starts the session, opens the PDO connection (`$db`), and loads the helpers. Form pages are named
for what they show (`login.php`, `edit.php`); handlers are `handle_*.php` or a verb (`store.php`,
`update.php`, `delete.php`).

The helpers (keep these names so every file reads the same):

| Helper | Does |
|---|---|
| `e($value)` | escape for HTML output |
| `redirect($url)` | `Location` header + `exit` |
| `require_post()` | 404 unless POST, then verifies the CSRF token (419 on failure) |
| `csrf_field()` | the hidden token input for a form |
| `check_login()` / `require_role('admin')` | guard a page |
| `current_user_id()` | the logged-in user's id for ownership queries |
| `back_with_errors($errors, $old, $url)` | flash errors + old input, go back |
| `old('field')` | refill a field after a failed submit |
| `__('key')` | translation from `inc/lang/{ar,en}.php` |
| `store_image($_FILES['image'])` | validate + optimize an upload → `[$fileName, $error]` |
| `delete_upload($fileName)` | remove an old upload safely |
| `not_found()` | 404 page + `exit` |

Ready, tested code for every file is in `references/snippets.md` — copy from there instead of
improvising.

## Request flow — Post/Redirect/Get

- **GET pages only read and render.** Anything that changes data (create, update, delete, toggle,
  logout) is a **POST** form with a CSRF field — never a GET link.
- A handle file: `require_post()` (method + CSRF) → `check_login()` if needed → read and `trim`
  input → validate into `$errors` → on errors: save errors + old input to the session and
  redirect back → otherwise write with a prepared statement → `$_SESSION['success']` → redirect.
- `redirect()` always ends with `exit`.
- Messages render from `inc/errors.php` / `inc/success.php` (or Toastr), then are unset.

## Database — PDO, prepared statements only

- One PDO connection (`$db`) from `inc/db.php`: `ERRMODE_EXCEPTION`, `FETCH_ASSOC`,
  `EMULATE_PREPARES => false`, charset `utf8mb4`.
- **Every query with a variable is a prepared statement** with `?` placeholders. Never build SQL
  with string concatenation or interpolation (`"... WHERE email='$email'"`) — that is SQL injection.
  Column/table names can't be bound: pick them from a hard-coded allow-list (e.g. sort columns).
- Scope user-owned rows by the logged-in user in the query itself:
  `WHERE id = ? AND user_id = ?` with `$_SESSION['user_id']` — never a `user_id` from the form.
- Multi-step writes that must succeed together (order + items, stock decrement) →
  `$db->beginTransaction()` … `commit()`, with `SELECT … FOR UPDATE` on the row being decremented;
  `rollBack()` in the `catch`.
- Schema: InnoDB, `utf8mb4_unicode_ci`, `INT UNSIGNED AUTO_INCREMENT` ids, real foreign keys,
  `UNIQUE` on emails/slugs, `created_at`/`updated_at` timestamps, snake_case column names
  (never `user-id`).

## Security — non-negotiable

- **Escape on output, not on input.** Store the trimmed raw value; print everything with `e()`
  (`htmlspecialchars(..., ENT_QUOTES, 'UTF-8')`). Don't run `htmlspecialchars` on input before
  saving — it corrupts the data and doesn't protect SQL.
- **CSRF:** a per-session token; `<?= csrf_field() ?>` in every POST form; `require_post()` verifies
  it with `hash_equals`.
- **Passwords:** `password_hash($password, PASSWORD_DEFAULT)` after validation passes;
  `password_verify()` on login. Never store or log plain passwords.
- **Login:** one generic message for wrong email *or* password ("Invalid email or password");
  `session_regenerate_id(true)` right after a successful login; logout clears and destroys the
  session (POST).
- **Session cookie:** `httponly`, `samesite=Lax`, `secure` when on HTTPS (set in `bootstrap.php`).
- **Authorization:** `check_login()` at the top of every protected page *and* handle file;
  `require_role('admin')` for admin pages. Hiding a link is not authorization.
- **Uploads:** check the upload error code and size; detect the real type with `finfo`
  (not the extension or `$_FILES['type']`); allow-list `image/jpeg|png|webp`; generate a random
  name; store in `uploads/` where PHP can't execute; delete the old file on update/delete
  (`basename()` the stored name).
- **Errors:** `display_errors` on only when `debug` is true in `config.php`; in production log
  instead. Never show SQL errors or stack traces to users. No leftover `echo`/`print_r`/`var_dump`.
- Never commit `config.php`, real credentials, or `uploads/` content.

## Validation

- Server-side always (client-side checks are only UX).
- Collect all problems into `$errors[]`, not just the first.
- `filter_var($email, FILTER_VALIDATE_EMAIL)`, `mb_strlen()` (counts Arabic correctly), `===`
  comparisons, `ctype_digit()` / `filter_var(..., FILTER_VALIDATE_INT)` for ids from the URL.
- Uniqueness (email) checked with a prepared `SELECT` before insert — and a `UNIQUE` index as the
  real guard.
- Keep old input (never the password) with `old('field')` after a failed submit.

## Image optimization — when users upload images that are shown on the site

Keep quality, shrink the size: `store_image()` validates the upload, then `optimize_image()`
resizes to a max width (~1600px), fixes phone-photo rotation, and re-encodes to **webp** quality ~80
(JPEG fallback if the server's GD has no webp). Re-encoding also strips EXIF/GPS data. Needs the GD
extension (`extension=gd`). Skip it for PDFs/documents — store those as-is with a random name.

## UI

- Bootstrap 5 from a pinned CDN version (or the project's existing version) in `inc/header.php`;
  vanilla JS for small interactions (confirm before delete, image preview, disable button on submit).
- Semantic HTML, a `<label for>` on every input, `alt` on images, visible focus.
- **Arabic/English:** `inc/lang/ar.php` and `inc/lang/en.php` return arrays (including `dir` and
  `lang`); `__('key')` prints a translation; `<html lang="<?= e(__('lang')) ?>" dir="<?= e(__('dir')) ?>">`;
  the language switcher accepts only an allow-list (`ar`, `en`). RTL uses `bootstrap.rtl.min.css`
  and `ms-`/`me-` utilities, never hardcoded left/right.

## Small JSON endpoints (optional)

For a tiny API next to the site (`api/*.php`): a shared `api/bootstrap.php` sets
`Content-Type: application/json`, checks the method (405 otherwise), reads JSON bodies with
`json_decode(file_get_contents('php://input'), true)`, and answers with
`json_response(['status' => …, 'message' => …, 'data' => …], $code)` — accurate codes (200/201/
401/404/405/422). Session-authenticated writes from the site's own JS send the CSRF token in an
`X-CSRF-Token` header. CORS only for the specific frontend origin from config, never `*`.
If the API grows beyond a few endpoints or needs tokens, move to Laravel (laravel-api-style).

## Testing before calling it done

Run it (`php -S localhost:8080` from the project root, or XAMPP) and click through: the happy path,
a validation error (old input kept), a missing/wrong CSRF token (rejected), another user's id in the
URL (not found), and logout. `php -l` on changed files.

## Official docs — when something isn't covered here

Read the official page before using a function you're unsure of, instead of relying on memory:

- PHP manual: `https://www.php.net/manual/en/` — e.g. `/book.pdo.php`, `/function.password-hash.php`,
  `/book.session.php`, `/function.htmlspecialchars.php`, `/book.image.php` (GD), `/book.fileinfo.php`.
- MySQL: `https://dev.mysql.com/doc/refman/8.0/en/` — e.g. `/innodb-locking-reads.html` (FOR UPDATE),
  `/create-index.html`.
- OWASP cheat sheets for security questions: `https://cheatsheetseries.owasp.org/`.

The docs win on how a function behaves; this skill wins on structure and style.
