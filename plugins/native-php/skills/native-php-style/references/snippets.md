# Native PHP snippets

A complete small app (Taskify: register, login, tasks CRUD with ownership and image upload, Arabic/English, a JSON endpoint). Every file below was run end to end on PHP 8.5 + MariaDB 10.4: validation errors with old input, CSRF rejection (419), generic login error, fake image rejected, a 3200x2000 3.1 MB JPEG stored as a 1600x1000 ~390 KB webp, another user's ids returning 404, and the old image deleted on update/delete.

Copy what the project needs and rename the entity (`tasks` → `products`…). Keep the helper names so every file reads the same. Not shown because they repeat a pattern: `db/.htaccess` (same as `inc/.htaccess`) and `auth/register.php` (the login form plus a name field — `old()` for name and email, never the password).

## Setup

### `.gitignore`

```gitignore
config.php
uploads/*
!uploads/.htaccess
```

### `.htaccess`

```apache
Options -Indexes
<FilesMatch "^config(\.example)?\.php$">
    Require all denied
</FilesMatch>
```

### `config.example.php`

```php
<?php
// Copy to config.php and fill in real values. config.php is never committed.
return [
    'debug' => true,                  // false on the live server
    'db' => [
        'host' => 'localhost',
        'name' => 'taskify',
        'user' => 'root',
        'pass' => '',
    ],
    'default_lang' => 'en',           // 'ar' or 'en'
    'api_allowed_origin' => '',       // e.g. 'https://app.example.com' — empty = no CORS
];
```

### `db/schema.sql`

```sql
CREATE DATABASE IF NOT EXISTS taskify CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE taskify;

CREATE TABLE users (
    id         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name       VARCHAR(100) NOT NULL,
    email      VARCHAR(191) NOT NULL UNIQUE,
    password   VARCHAR(255) NOT NULL,
    role       ENUM('user', 'admin') NOT NULL DEFAULT 'user',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE tasks (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     INT UNSIGNED NOT NULL,
    title       VARCHAR(150) NOT NULL,
    description TEXT NOT NULL,
    image       VARCHAR(100) NULL,
    status      ENUM('pending', 'done') NOT NULL DEFAULT 'pending',
    created_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_tasks_user FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE CASCADE,
    INDEX idx_tasks_user_status (user_id, status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

## inc/ — loaded by every page

### `inc/bootstrap.php`

```php
<?php
declare(strict_types=1);

$config = require __DIR__ . '/../config.php';

error_reporting(E_ALL);
ini_set('display_errors', $config['debug'] ? '1' : '0');
ini_set('log_errors', '1');

session_set_cookie_params([
    'httponly' => true,
    'samesite' => 'Lax',
    'secure'   => !empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off',
]);
session_start();

require __DIR__ . '/db.php';
require __DIR__ . '/functions.php';
require __DIR__ . '/csrf.php';
require __DIR__ . '/auth.php';
```

### `inc/db.php`

```php
<?php
$db = new PDO(
    "mysql:host={$config['db']['host']};dbname={$config['db']['name']};charset=utf8mb4",
    $config['db']['user'],
    $config['db']['pass'],
    [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        PDO::ATTR_EMULATE_PREPARES   => false,
    ]
);
```

### `inc/functions.php`

```php
<?php
function e(?string $value): string
{
    return htmlspecialchars($value ?? '', ENT_QUOTES, 'UTF-8');
}

function redirect(string $url): never
{
    header("Location: $url");
    exit;
}

function not_found(): never
{
    http_response_code(404);
    require __DIR__ . '/404.php';
    exit;
}

// Only POST requests with a valid CSRF token get past this line.
function require_post(): void
{
    if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
        not_found();
    }
    verify_csrf();
}

// Save errors + old input and go back to the form.
function back_with_errors(array $errors, array $old, string $url): never
{
    $_SESSION['errors'] = $errors;
    $_SESSION['old'] = $old;
    redirect($url);
}

function old(string $key): string
{
    return (string) ($_SESSION['old'][$key] ?? '');
}

function __(string $key): string
{
    static $lines = null;
    if ($lines === null) {
        global $config;
        $lang = $_SESSION['lang'] ?? $config['default_lang'];
        $lines = require __DIR__ . "/lang/$lang.php";
    }
    return $lines[$key] ?? $key;
}

function delete_upload(?string $fileName): void
{
    if (!$fileName) {
        return;
    }
    $path = __DIR__ . '/../uploads/' . basename($fileName);
    if (is_file($path)) {
        unlink($path);
    }
}

/**
 * Validates an uploaded image and saves an optimized copy in uploads/.
 * Returns [fileName, null] on success or [null, errorMessage].
 */
function store_image(array $file, int $maxMb = 5): array
{
    if ($file['error'] === UPLOAD_ERR_NO_FILE) {
        return [null, 'Image is required'];
    }
    if ($file['error'] !== UPLOAD_ERR_OK) {
        return [null, 'Image upload failed'];
    }
    if ($file['size'] > $maxMb * 1024 * 1024) {
        return [null, "Image must be at most {$maxMb} MB"];
    }

    // The real type from the file's content — never trust the extension or $_FILES['type'].
    $mime = (new finfo(FILEINFO_MIME_TYPE))->file($file['tmp_name']);
    if (!in_array($mime, ['image/jpeg', 'image/png', 'image/webp'], true)) {
        return [null, 'Only JPG, PNG or WEBP images are allowed'];
    }

    return [optimize_image($file['tmp_name'], $mime, __DIR__ . '/../uploads'), null];
}

// Resize to a max width, fix phone-photo rotation, re-encode as webp (strips EXIF/GPS).
function optimize_image(string $tmpPath, string $mime, string $dir, int $maxWidth = 1600, int $quality = 80): string
{
    $image = match ($mime) {
        'image/jpeg' => imagecreatefromjpeg($tmpPath),
        'image/png'  => imagecreatefrompng($tmpPath),
        'image/webp' => imagecreatefromwebp($tmpPath),
    };

    if ($mime === 'image/jpeg' && function_exists('exif_read_data')) {
        $orientation = @exif_read_data($tmpPath)['Orientation'] ?? 1;
        $image = match ($orientation) {
            3 => imagerotate($image, 180, 0),
            6 => imagerotate($image, -90, 0),
            8 => imagerotate($image, 90, 0),
            default => $image,
        };
    }

    if (imagesx($image) > $maxWidth) {
        $image = imagescale($image, $maxWidth);
    }

    $name = bin2hex(random_bytes(16));
    if (function_exists('imagewebp')) {
        imagepalettetotruecolor($image);
        imagealphablending($image, false);
        imagesavealpha($image, true);
        imagewebp($image, "$dir/$name.webp", $quality);
        return "$name.webp";
    }

    imagejpeg($image, "$dir/$name.jpg", $quality);
    return "$name.jpg";
}
```

### `inc/csrf.php`

```php
<?php
function csrf_token(): string
{
    if (empty($_SESSION['csrf_token'])) {
        $_SESSION['csrf_token'] = bin2hex(random_bytes(32));
    }
    return $_SESSION['csrf_token'];
}

function csrf_field(): string
{
    return '<input type="hidden" name="csrf_token" value="' . csrf_token() . '">';
}

function verify_csrf(): void
{
    $sent = $_POST['csrf_token'] ?? '';
    if (!is_string($sent) || !hash_equals(csrf_token(), $sent)) {
        http_response_code(419);
        exit('Page expired. Go back, refresh the page and try again.');
    }
}
```

### `inc/auth.php`

```php
<?php
function check_login(): void
{
    if (!isset($_SESSION['user_id'])) {
        redirect('../auth/login.php');
    }
}

function require_role(string $role): void
{
    check_login();
    if (($_SESSION['user_role'] ?? '') !== $role) {
        not_found();
    }
}

function current_user_id(): int
{
    return (int) $_SESSION['user_id'];
}
```

## Layout partials

### `inc/header.php`

```php
<!doctype html>
<html lang="<?= e(__('lang')) ?>" dir="<?= e(__('dir')) ?>">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><?= e($title ?? 'Taskify') ?></title>
    <?php if (__('dir') === 'rtl'): ?>
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.rtl.min.css">
    <?php else: ?>
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css">
    <?php endif; ?>
    <link rel="stylesheet" href="../assets/css/style.css">
</head>
<body>
<nav class="navbar bg-body-tertiary mb-4">
    <div class="container">
        <a class="navbar-brand" href="../tasks/index.php">Taskify</a>
        <div class="d-flex gap-2 align-items-center">
            <form action="../handle_lang.php" method="post" class="m-0">
                <?= csrf_field() ?>
                <input type="hidden" name="lang" value="<?= __('lang') === 'ar' ? 'en' : 'ar' ?>">
                <button type="submit" class="btn btn-link btn-sm"><?= __('lang') === 'ar' ? 'English' : 'العربية' ?></button>
            </form>
            <?php if (isset($_SESSION['user_id'])): ?>
                <span class="text-body-secondary"><?= e($_SESSION['user_name']) ?></span>
                <form action="../auth/logout.php" method="post" class="m-0">
                    <?= csrf_field() ?>
                    <button type="submit" class="btn btn-outline-secondary btn-sm"><?= e(__('Logout')) ?></button>
                </form>
            <?php endif; ?>
        </div>
    </div>
</nav>
```

### `inc/footer.php`

```php
<script>
    // Confirm destructive actions, then block double submits.
    document.querySelectorAll('form').forEach(form => {
        form.addEventListener('submit', event => {
            if (form.dataset.confirm && !confirm(form.dataset.confirm)) {
                event.preventDefault();
                return;
            }
            form.querySelectorAll('button[type=submit]').forEach(button => button.disabled = true);
        });
    });
</script>
</body>
</html>
<?php unset($_SESSION['old']); ?>
```

### `inc/errors.php`

```php
<?php if (!empty($_SESSION['errors'])): ?>
    <div class="alert alert-danger" role="alert">
        <ul class="mb-0">
            <?php foreach ($_SESSION['errors'] as $error): ?>
                <li><?= e($error) ?></li>
            <?php endforeach; ?>
        </ul>
    </div>
    <?php unset($_SESSION['errors']); ?>
<?php endif; ?>
```

### `inc/success.php`

```php
<?php if (!empty($_SESSION['success'])): ?>
    <div class="alert alert-success" role="status"><?= e($_SESSION['success']) ?></div>
    <?php unset($_SESSION['success']); ?>
<?php endif; ?>
```

### `inc/404.php`

```php
<?php require __DIR__ . '/header.php'; ?>
<main class="container py-5 text-center">
    <h1 class="h3">404</h1>
    <p><?= e(__('Page not found')) ?></p>
</main>
<?php require __DIR__ . '/footer.php'; ?>
```

### `inc/lang/ar.php`

```php
<?php
return [
    'lang' => 'ar',
    'dir' => 'rtl',
    'Logout' => 'تسجيل الخروج',
    'My tasks' => 'مهامي',
    'Page not found' => 'الصفحة غير موجودة',
];
```

### `inc/lang/en.php`

```php
<?php
return [
    'lang' => 'en',
    'dir' => 'ltr',
    'Logout' => 'Logout',
    'My tasks' => 'My tasks',
    'Page not found' => 'Page not found',
];
```

### `handle_lang.php`

```php
<?php
require_once __DIR__ . '/inc/bootstrap.php';
require_post();

$lang = $_POST['lang'] ?? '';
if (in_array($lang, ['ar', 'en'], true)) {
    $_SESSION['lang'] = $lang;
}

// Back to the same page, but only a path on this site (never an outside URL).
$back = parse_url($_SERVER['HTTP_REFERER'] ?? '', PHP_URL_PATH) ?: 'tasks/index.php';
redirect($back);
```

## Auth

### `auth/login.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
$title = 'Login';
require __DIR__ . '/../inc/header.php';
?>
<main class="container" style="max-width: 480px">
    <h1 class="h4 mb-3">Login</h1>
    <?php require __DIR__ . '/../inc/success.php'; ?>
    <?php require __DIR__ . '/../inc/errors.php'; ?>
    <form action="handle_login.php" method="post">
        <?= csrf_field() ?>
        <div class="mb-3">
            <label for="email" class="form-label">Email</label>
            <input id="email" name="email" type="email" class="form-control" value="<?= e(old('email')) ?>" required>
        </div>
        <div class="mb-3">
            <label for="password" class="form-label">Password</label>
            <input id="password" name="password" type="password" class="form-control" required>
        </div>
        <button type="submit" class="btn btn-primary w-100">Login</button>
        <p class="mt-3 text-center"><a href="register.php">Create an account</a></p>
    </form>
</main>
<?php require __DIR__ . '/../inc/footer.php'; ?>
```

### `auth/handle_login.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
require_post();

$email = strtolower(trim($_POST['email'] ?? ''));
$password = $_POST['password'] ?? '';

if ($email === '' || $password === '') {
    back_with_errors(['Email and password are required'], ['email' => $email], 'login.php');
}

$stmt = $db->prepare('SELECT id, name, password, role FROM users WHERE email = ?');
$stmt->execute([$email]);
$user = $stmt->fetch();

// One message for both cases, so the form doesn't reveal which emails exist.
if (!$user || !password_verify($password, $user['password'])) {
    back_with_errors(['Invalid email or password'], ['email' => $email], 'login.php');
}

session_regenerate_id(true);
$_SESSION['user_id'] = $user['id'];
$_SESSION['user_name'] = $user['name'];
$_SESSION['user_role'] = $user['role'];
$_SESSION['success'] = "Welcome {$user['name']}";
redirect('../tasks/index.php');
```

### `auth/handle_register.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
require_post();

$name = trim($_POST['name'] ?? '');
$email = strtolower(trim($_POST['email'] ?? ''));
$password = $_POST['password'] ?? '';
$errors = [];

if ($name === '') {
    $errors[] = 'Name is required';
} elseif (mb_strlen($name) < 3 || mb_strlen($name) > 100) {
    $errors[] = 'Name must be between 3 and 100 characters';
}

if ($email === '') {
    $errors[] = 'Email is required';
} elseif (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
    $errors[] = 'Invalid email format';
} else {
    $stmt = $db->prepare('SELECT id FROM users WHERE email = ?');
    $stmt->execute([$email]);
    if ($stmt->fetch()) {
        $errors[] = 'Email already exists';
    }
}

if (mb_strlen($password) < 8) {
    $errors[] = 'Password must be at least 8 characters';
}

if ($errors) {
    back_with_errors($errors, ['name' => $name, 'email' => $email], 'register.php');
}

$stmt = $db->prepare('INSERT INTO users (name, email, password) VALUES (?, ?, ?)');
$stmt->execute([$name, $email, password_hash($password, PASSWORD_DEFAULT)]);

$_SESSION['success'] = 'Account created, you can log in now';
redirect('login.php');
```

### `auth/logout.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
require_post();

$_SESSION = [];
session_destroy();
redirect('login.php');
```

## CRUD with ownership and an optional image (tasks/)

### `tasks/index.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
check_login();

$stmt = $db->prepare('SELECT id, title, description, image, status FROM tasks WHERE user_id = ? ORDER BY id DESC');
$stmt->execute([current_user_id()]);
$tasks = $stmt->fetchAll();

$title = __('My tasks');
require __DIR__ . '/../inc/header.php';
?>
<main class="container">
    <h1 class="h4 mb-3"><?= e(__('My tasks')) ?></h1>
    <?php require __DIR__ . '/../inc/success.php'; ?>
    <?php require __DIR__ . '/../inc/errors.php'; ?>

    <form action="store.php" method="post" enctype="multipart/form-data" class="card card-body mb-4">
        <?= csrf_field() ?>
        <div class="mb-3">
            <label for="title" class="form-label">Title</label>
            <input id="title" name="title" class="form-control" value="<?= e(old('title')) ?>" required>
        </div>
        <div class="mb-3">
            <label for="description" class="form-label">Description</label>
            <textarea id="description" name="description" class="form-control" rows="2" required><?= e(old('description')) ?></textarea>
        </div>
        <div class="mb-3">
            <label for="image" class="form-label">Image (optional)</label>
            <input id="image" name="image" type="file" class="form-control" accept="image/jpeg,image/png,image/webp">
        </div>
        <button type="submit" class="btn btn-primary">Add task</button>
    </form>

    <?php if (!$tasks): ?>
        <p class="text-body-secondary">No tasks yet.</p>
    <?php endif; ?>

    <ul class="list-group">
        <?php foreach ($tasks as $task): ?>
            <li class="list-group-item d-flex gap-3 align-items-center">
                <?php if ($task['image']): ?>
                    <img src="../uploads/<?= e($task['image']) ?>" alt="<?= e($task['title']) ?>" width="64" height="64" class="rounded object-fit-cover">
                <?php endif; ?>
                <div class="flex-grow-1">
                    <strong class="<?= $task['status'] === 'done' ? 'text-decoration-line-through' : '' ?>"><?= e($task['title']) ?></strong>
                    <span class="badge text-bg-<?= $task['status'] === 'done' ? 'success' : 'secondary' ?>"><?= e($task['status']) ?></span>
                    <div class="small text-body-secondary"><?= e($task['description']) ?></div>
                </div>
                <form action="toggle.php" method="post" class="m-0">
                    <?= csrf_field() ?>
                    <input type="hidden" name="id" value="<?= $task['id'] ?>">
                    <button type="submit" class="btn btn-sm btn-outline-success">Toggle</button>
                </form>
                <a href="edit.php?id=<?= $task['id'] ?>" class="btn btn-sm btn-outline-primary">Edit</a>
                <form action="delete.php" method="post" class="m-0" data-confirm="Delete this task?">
                    <?= csrf_field() ?>
                    <input type="hidden" name="id" value="<?= $task['id'] ?>">
                    <button type="submit" class="btn btn-sm btn-outline-danger">Delete</button>
                </form>
            </li>
        <?php endforeach; ?>
    </ul>
</main>
<?php require __DIR__ . '/../inc/footer.php'; ?>
```

### `tasks/store.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
require_post();
check_login();

$title = trim($_POST['title'] ?? '');
$description = trim($_POST['description'] ?? '');
$errors = [];

if (mb_strlen($title) < 3 || mb_strlen($title) > 150) {
    $errors[] = 'Title must be between 3 and 150 characters';
}
if ($description === '') {
    $errors[] = 'Description is required';
}

$image = null;
if (!$errors && ($_FILES['image']['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_NO_FILE) {
    [$image, $imageError] = store_image($_FILES['image']);
    if ($imageError) {
        $errors[] = $imageError;
    }
}

if ($errors) {
    back_with_errors($errors, ['title' => $title, 'description' => $description], 'index.php');
}

$stmt = $db->prepare('INSERT INTO tasks (user_id, title, description, image) VALUES (?, ?, ?, ?)');
$stmt->execute([current_user_id(), $title, $description, $image]);

$_SESSION['success'] = 'Task added successfully';
redirect('index.php');
```

### `tasks/edit.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
check_login();

$id = filter_var($_GET['id'] ?? '', FILTER_VALIDATE_INT);
$stmt = $db->prepare('SELECT id, title, description, image FROM tasks WHERE id = ? AND user_id = ?');
$stmt->execute([$id, current_user_id()]);
$task = $stmt->fetch();

if (!$task) {
    not_found();
}

$title = 'Edit task';
require __DIR__ . '/../inc/header.php';
?>
<main class="container" style="max-width: 640px">
    <h1 class="h4 mb-3">Edit task</h1>
    <?php require __DIR__ . '/../inc/errors.php'; ?>
    <form action="update.php" method="post" enctype="multipart/form-data">
        <?= csrf_field() ?>
        <input type="hidden" name="id" value="<?= $task['id'] ?>">
        <div class="mb-3">
            <label for="title" class="form-label">Title</label>
            <input id="title" name="title" class="form-control" value="<?= e(old('title') ?: $task['title']) ?>" required>
        </div>
        <div class="mb-3">
            <label for="description" class="form-label">Description</label>
            <textarea id="description" name="description" class="form-control" rows="3" required><?= e(old('description') ?: $task['description']) ?></textarea>
        </div>
        <div class="mb-3">
            <label for="image" class="form-label">Replace image (optional)</label>
            <input id="image" name="image" type="file" class="form-control" accept="image/jpeg,image/png,image/webp">
            <?php if ($task['image']): ?>
                <img src="../uploads/<?= e($task['image']) ?>" alt="<?= e($task['title']) ?>" class="img-thumbnail mt-2" width="160">
            <?php endif; ?>
        </div>
        <button type="submit" class="btn btn-primary">Save</button>
        <a href="index.php" class="btn btn-link">Cancel</a>
    </form>
</main>
<?php require __DIR__ . '/../inc/footer.php'; ?>
```

### `tasks/update.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
require_post();
check_login();

$id = filter_var($_POST['id'] ?? '', FILTER_VALIDATE_INT);
$stmt = $db->prepare('SELECT id, image FROM tasks WHERE id = ? AND user_id = ?');
$stmt->execute([$id, current_user_id()]);
$task = $stmt->fetch();

if (!$task) {
    not_found();
}

$title = trim($_POST['title'] ?? '');
$description = trim($_POST['description'] ?? '');
$errors = [];

if (mb_strlen($title) < 3 || mb_strlen($title) > 150) {
    $errors[] = 'Title must be between 3 and 150 characters';
}
if ($description === '') {
    $errors[] = 'Description is required';
}

$image = $task['image'];
if (!$errors && ($_FILES['image']['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_NO_FILE) {
    [$newImage, $imageError] = store_image($_FILES['image']);
    if ($imageError) {
        $errors[] = $imageError;
    } else {
        delete_upload($task['image']);
        $image = $newImage;
    }
}

if ($errors) {
    back_with_errors($errors, ['title' => $title, 'description' => $description], "edit.php?id={$task['id']}");
}

$stmt = $db->prepare('UPDATE tasks SET title = ?, description = ?, image = ? WHERE id = ? AND user_id = ?');
$stmt->execute([$title, $description, $image, $task['id'], current_user_id()]);

$_SESSION['success'] = 'Task updated successfully';
redirect('index.php');
```

### `tasks/toggle.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
require_post();
check_login();

$id = filter_var($_POST['id'] ?? '', FILTER_VALIDATE_INT);
$stmt = $db->prepare(
    "UPDATE tasks SET status = IF(status = 'pending', 'done', 'pending') WHERE id = ? AND user_id = ?"
);
$stmt->execute([$id, current_user_id()]);

if ($stmt->rowCount() === 0) {
    not_found();
}

$_SESSION['success'] = 'Task status updated';
redirect('index.php');
```

### `tasks/delete.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';
require_post();
check_login();

$id = filter_var($_POST['id'] ?? '', FILTER_VALIDATE_INT);
$stmt = $db->prepare('SELECT id, image FROM tasks WHERE id = ? AND user_id = ?');
$stmt->execute([$id, current_user_id()]);
$task = $stmt->fetch();

if (!$task) {
    not_found();
}

$db->prepare('DELETE FROM tasks WHERE id = ? AND user_id = ?')->execute([$task['id'], current_user_id()]);
delete_upload($task['image']);

$_SESSION['success'] = 'Task deleted successfully';
redirect('index.php');
```

## Optional: small JSON API (api/)

### `api/bootstrap.php`

```php
<?php
require_once __DIR__ . '/../inc/bootstrap.php';

header('Content-Type: application/json; charset=UTF-8');

if ($config['api_allowed_origin'] !== '') {
    header("Access-Control-Allow-Origin: {$config['api_allowed_origin']}");
    header('Vary: Origin');
}

function json_response(array $body, int $status = 200): never
{
    http_response_code($status);
    echo json_encode($body, JSON_UNESCAPED_UNICODE);
    exit;
}

function allow_methods(string ...$methods): void
{
    if (!in_array($_SERVER['REQUEST_METHOD'], $methods, true)) {
        header('Allow: ' . implode(', ', $methods));
        json_response(['status' => false, 'message' => 'Method not allowed'], 405);
    }
}

function json_input(): array
{
    return json_decode(file_get_contents('php://input'), true) ?: [];
}

function api_require_login(): void
{
    if (!isset($_SESSION['user_id'])) {
        json_response(['status' => false, 'message' => 'Unauthenticated'], 401);
    }
}

// Same-site JS sends the page's CSRF token in a header on every write.
function api_verify_csrf(): void
{
    $sent = $_SERVER['HTTP_X_CSRF_TOKEN'] ?? '';
    if (!hash_equals(csrf_token(), $sent)) {
        json_response(['status' => false, 'message' => 'Invalid CSRF token'], 419);
    }
}
```

### `api/tasks.php`

```php
<?php
require_once __DIR__ . '/bootstrap.php';
allow_methods('GET', 'POST');
api_require_login();

if ($_SERVER['REQUEST_METHOD'] === 'GET') {
    $stmt = $db->prepare('SELECT id, title, description, status FROM tasks WHERE user_id = ? ORDER BY id DESC');
    $stmt->execute([current_user_id()]);
    json_response(['status' => true, 'message' => 'Tasks', 'data' => $stmt->fetchAll()]);
}

api_verify_csrf();
$input = json_input();
$title = trim((string) ($input['title'] ?? ''));
$description = trim((string) ($input['description'] ?? ''));
$errors = [];

if (mb_strlen($title) < 3 || mb_strlen($title) > 150) {
    $errors['title'] = 'Title must be between 3 and 150 characters';
}
if ($description === '') {
    $errors['description'] = 'Description is required';
}
if ($errors) {
    json_response(['status' => false, 'message' => 'Validation error', 'errors' => $errors], 422);
}

$stmt = $db->prepare('INSERT INTO tasks (user_id, title, description) VALUES (?, ?, ?)');
$stmt->execute([current_user_id(), $title, $description]);

json_response([
    'status' => true,
    'message' => 'Task created successfully',
    'data' => ['id' => (int) $db->lastInsertId(), 'title' => $title, 'description' => $description, 'status' => 'pending'],
], 201);
```

## Protecting folders on Apache / LiteSpeed

### `inc/.htaccess`

```apache
Require all denied
```

### `uploads/.htaccess`

```apache
# Uploaded files are data, never code.
<FilesMatch "\.(php\d?|phtml|phar)$">
    Require all denied
</FilesMatch>
Options -Indexes
```

## Transactions — decrementing stock

```php
try {
    $db->beginTransaction();

    // Lock the product row so two orders can't take the last item.
    $stmt = $db->prepare('SELECT id, stock, price FROM products WHERE id = ? FOR UPDATE');
    $stmt->execute([$productId]);
    $product = $stmt->fetch();

    if (!$product || $product['stock'] < $quantity) {
        $db->rollBack();
        back_with_errors(['Not enough stock'], [], '../products/index.php');
    }

    $db->prepare('UPDATE products SET stock = stock - ? WHERE id = ?')->execute([$quantity, $product['id']]);
    $db->prepare('INSERT INTO orders (user_id, product_id, quantity, total) VALUES (?, ?, ?, ?)')
       ->execute([current_user_id(), $product['id'], $quantity, $product['price'] * $quantity]);

    $db->commit();
} catch (Throwable $e) {
    if ($db->inTransaction()) {
        $db->rollBack();
    }
    throw $e;
}
```
## Dynamic sorting — allow-list column names

```php
// Column names can't be bound with ?, so pick them from a fixed list.
$sortable = ['title' => 'title', 'newest' => 'id DESC'];
$order = $sortable[$_GET['sort'] ?? ''] ?? 'id DESC';
$stmt = $db->prepare("SELECT id, title FROM tasks WHERE user_id = ? ORDER BY $order");
$stmt->execute([current_user_id()]);
```
