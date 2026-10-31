# Laravel troubleshooting

Check the log first: `storage/logs/laravel.log` (`tail -n 50`). Then match the symptom.

| Symptom | Usual cause | Fix |
|---|---|---|
| `SQLSTATE[HY000] [2002] Connection refused` | MySQL not running, or wrong `DB_HOST` | Start MySQL; in Docker `DB_HOST=mysql` |
| `Access denied for user` | Wrong DB user/password in `.env` | Fix them, then `php artisan optimize:clear` |
| `No application encryption key` | Empty `APP_KEY` | `php artisan key:generate` (never change it on a live site) |
| `Target class [X] does not exist` / `Class not found` | Namespace doesn't match the folder, or a missing `use` | Fix the namespace (PSR-4), `composer dump-autoload` |
| `Add [x] to fillable property` | Field missing from `$fillable` | Add it to `$fillable` (never `$guarded = []`) |
| `SQLSTATE[23000] ... 1452` | Foreign key points to a missing row | Add an `exists:table,id` rule |
| `SQLSTATE[23000] ... 1451` | Deleting a parent that still has children (`restrictOnDelete`) | Intended — tell the user to move/delete the children first |
| `SQLSTATE[42S22] Column not found` | Typo, or migration not run | `php artisan migrate:status` |
| `Attempt to read property on null` | Missing record or empty relation | `findOrFail()`, or `?->` for an optional relation |
| `Attempted to lazy load [x]` | `preventLazyLoading` caught an N+1 | Add `with('x')` / `load('x')` |
| A relation is missing from the output | Not eager-loaded (`whenLoaded` hides it) | `with()` in the query or `load()` after route binding |
| `Allowed memory size exhausted` | Loading too many rows at once | `chunkById()` / `lazyById()` / paginate |
| `419 Page Expired` | POST without `@csrf`, or the session expired | `@csrf` in the form; for JSON clients use `routes/api.php` |
| `Route [login] not defined` (API) | Unauthenticated request that isn't JSON | Send `Accept: application/json`; render JSON for `api/*` in `bootstrap/app.php` |
| `Unauthenticated.` | Missing/expired Bearer token | Log in again; send `Authorization: Bearer <token>` |
| `This action is unauthorized.` | A Policy or FormRequest `authorize()` returned false | Check the policy/ownership rule |
| Uploaded images 404 | No storage link, or wrong `APP_URL` | `php artisan storage:link` (or a manual symlink on hosts that block it), fix `APP_URL` |
| `env()` returns null only on the server | Config is cached | Use `env()` only in `config/*.php`, `config()` elsewhere; re-run `php artisan optimize` after editing `.env` |
| `laravel.log could not be opened` / 500 after upload | Write permissions | `chmod -R 775 storage bootstrap/cache` |
| Old data still shows | A cache key wasn't cleared | `Cache::forget()` the key when data changes |
| Jobs never run | No worker, or a worker running old code | `php artisan queue:work`, `queue:restart` after deploy; locally `QUEUE_CONNECTION=sync` |
| CORS error in the browser | Frontend domain not allowed | `config/cors.php` (`php artisan config:publish cors`) — list the domain, not `*` in production |
| 404 on every API route | `routes/api.php` not installed, or URL missing `/api` | `php artisan install:api` |
| Conversions (webp) not generated | GD/Imagick missing, or conversions queued without a worker | Enable the extension; use `nonQueued()` or run a worker |

## Useful commands

```bash
php artisan route:list --path=admin     # what routes exist
php artisan migrate:status              # which migrations ran
php artisan optimize:clear              # clear config/route/view/cache
php artisan about                       # versions + environment summary
composer audit                          # known vulnerabilities in dependencies
```
