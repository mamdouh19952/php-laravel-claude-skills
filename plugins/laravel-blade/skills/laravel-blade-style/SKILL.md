---
name: laravel-blade-style
description: 'Conventions for server-rendered Laravel projects — Blade templates, Bootstrap, HTML/CSS, and vanilla JS (jQuery only for Toastr) — for admin panels, dashboards, catalogs, stores, and small business systems. Use whenever working in a Laravel project that renders Blade views — adding a page, a CRUD screen, a web route, a web controller, a Blade layout/component/partial, a form, auth pages, Toastr notifications, the loader, file/image uploads, or small JS interactions — even if the skill isn''t named. Not for JSON APIs (use laravel-api-style) or Vue/React SPAs. Trigger words: Blade, Bootstrap, web.php, view, layout, partial, Blade component, admin panel, dashboard, CRUD, form, Toastr, jQuery, لارافيل بليد.'
---

# Laravel Blade style

Server-rendered, simple, web-only. The goal is code that is clean, boring in the good sense, and
easy to maintain — as if one person wrote all of it. **Match whatever already exists in the
project** — don't introduce patterns, packages, or layers that aren't already there or genuinely
needed.

## Before writing code

1. Read the project's `CLAUDE.md` (if any) for versions, language/RTL, auth, user areas, media.
   If it's missing and the answer matters, ask: Laravel/PHP/Bootstrap versions, Arabic/English/
   bilingual, which user areas (admin / user), whether there are image uploads.
2. Inspect existing routes, controllers, models, FormRequests, and Blade layouts/components/partials.
3. Check `composer.json` / `package.json` before suggesting any package.
4. For a multi-step feature, state a short plan before making a large change.

## Baseline rules (always)

- **Security:** validate everything server-side; `@csrf` on every state-changing form (never
  disable CSRF to "make it work"); authorize on the server — hiding a button is not authorization;
  never trust a client-sent `id`/`user_id` for ownership — derive it from `auth()->user()`; never
  show stack traces, SQL errors, or secrets to users; never commit `.env`, `vendor/`,
  `node_modules/`.
- **Tests:** feature tests for user-facing flows — at minimum the happy path, the validation-failure
  path, and the auth/ownership-failure path when one exists. Don't test framework behavior.
- **Change discipline:** don't install packages, restructure folders, or change existing behavior
  without checking in first. Don't touch unrelated code.
- **Comments are opt-in:** only when the *why* isn't obvious (a lock, a transaction, a workaround).

## Keep it simple — no over-engineering

- **Controllers hold simple CRUD logic.** Add a **Service** only when logic is complex, reused
  across controllers (or by a command/job), transactional across multiple models, or hurting
  readability. Never a pass-through Service that only forwards to Eloquent.
- **Repository only** when the data source itself genuinely needs to be swappable — rare.
- **No Events/Listeners/Jobs/Queues** for ordinary CRUD — only for real async work (mail, SMS,
  heavy processing).
- When you do add a Service/Repository/Policy, say in one line why it's justified.
- If the project already has Services/Repositories/Events, follow them; don't rewrite them.

## Tech stack

- Laravel, PHP, Eloquent, MySQL.
- Blade + **Bootstrap** (follow the version the project already uses).
- **Vanilla JS** for project code. jQuery is present only because Toastr needs it — don't write
  new jQuery unless the project already uses it.
- **No SPA, no Vue/React, no Livewire, no frontend state libraries.**
- Assets: the project's existing Vite or `public/` setup — don't change the pipeline.

## Architecture & routes

- **Web-only by default.** Routes in `routes/web.php`; controllers return views or redirects.
  Don't create `routes/api.php`, API Resources, or JSON endpoints unless asked or an AJAX feature
  genuinely needs one.
- Default flow: **Form → route → controller → DB → redirect → Blade** (Post/Redirect/Get).
- `Route::resource()` for standard CRUD — not hand-made URIs like `/deleteProduct/{id}`.
- Route names: `{area}.{resource}.{action}` — e.g. `admin.products.index`, `admin.products.store`.
- One `auth` middleware plus one `admin` middleware (`EnsureUserIsAdmin`) on the admin group —
  not several overlapping auth/admin/user middlewares.

## Controllers

- FormRequest for validation; `$request->validated()` for create/update — never inline
  `$request->validate()` or `Validator::make()` in a controller.
- Route model binding where practical; `findOrFail()` instead of `find()` + null check.
- GET → view; POST/PUT/PATCH/DELETE → redirect with a flash message:
  ```php
  return redirect()->route('admin.products.index')->with('success', __('Product created successfully'));
  ```
- No markup in controllers.

## Validation

- Rules in the FormRequest's `rules()`; `authorize()` returns `true` unless the request itself
  checks ownership.
- Unique on update: `Rule::unique('products', 'slug')->ignore($this->route('product'))`.
- Normalize input before validating with `prepareForValidation()` (e.g. build a slug, trim).
- Passwords: `Password::min(8)`; the User model casts `password` to `hashed`.
- Foreign keys: `exists:categories,id` so a bad id is a 422, not a SQL error.

## Models & migrations

- Explicit `$fillable` — never `$guarded = []`. `$hidden` for sensitive columns.
- `casts()` for dates, booleans, enums, `hashed`.
- Relationships defined on both sides.
- Foreign keys: `restrictOnDelete()` by default for important parents (a category with products,
  a product with orders); `cascadeOnDelete()` only for children that mean nothing without the
  parent (order items).

## Blade

- Presentation only — no queries or writes in Blade, no business logic.
- `{{ }}` by default; `{!! !!}` only for trusted HTML.
- Every state-changing form: `@csrf`, plus `@method('PUT'|'PATCH'|'DELETE')` where needed.
  Deletes are forms (POST + `@method('DELETE')`), never GET links.
- Preserve input with `old()`; show validation errors with one shared pattern.
- Layouts, partials, and Blade components for reused UI (input, select, button, modal, card,
  table, pagination). Don't make a component for every tiny fragment.

Default view structure for new projects:

```
resources/views/
├── layouts/      app.blade.php, guest.blade.php
├── components/   input, button, alert, loader, ...
├── partials/     toast.blade.php, scripts.blade.php
├── admin/
└── user/
```

Use area folders (`admin/`, `user/`) only when the project has multiple user areas.

## JavaScript

Small, local, easy to read: confirm dialogs, modals, show/hide, tabs, image preview before upload,
preventing double submit, date pickers.

- Don't create API endpoints just to avoid a page reload.
- Don't duplicate Laravel business logic in JS.
- **AJAX is opt-in** — only when asked or the server-rendered flow genuinely can't deliver the UX.
  Then: `fetch()` (or `$.ajax()` in jQuery projects), send the CSRF token
  (`<meta name="csrf-token">` → `X-CSRF-TOKEN` header), and return JSON only from the endpoints
  that need it.

## Auth

- New projects: **`laravel/ui`** with Bootstrap — `composer require laravel/ui` →
  `php artisan ui bootstrap --auth`; routes via `Auth::routes(['verify' => true])` when email
  verification is wanted.
- The auth views use the project layouts — no second, generated layout.
- Redirect after login/register with `$redirectTo` on the auth controllers.
- Throttle login (a named `login` rate limiter: 5/minute per email + IP).
- **Never Jetstream, Livewire, or Fortify** in these projects — they bring a second stack to maintain.

## Authorization & ownership

- Middleware for simple role gates (`admin`); a **Policy** when the rule is per-record ("only the
  owner can edit") or has more than one condition.
- Load user-owned records through the user, so another user's id can't be guessed (IDOR):
  ```php
  $order = auth()->user()->orders()->findOrFail($id);
  ```

## Notifications (toasts)

- Use the project's existing system; don't add a second one.
- New projects: **Toastr + jQuery** from pinned versions (never `latest`), loaded in
  `partials/scripts.blade.php`.
- One shared `partials/toast.blade.php` reads `session('success'|'error'|'warning'|'info')` and
  passes each message with `@json(...)` — e.g. `toastr.success(@json(session('success')))`.
  Controllers only set flash messages.

## Loader

- `<x-loader />` (a Bootstrap spinner overlay) sits once in the layout.
- It shows on every form submit unless the form has `data-no-loader`, and hides again on
  `pageshow` (so it isn't stuck after the browser's back button).

## File & image uploads

- Default: `spatie/laravel-medialibrary` wrapped in `app/Services/MediaService.php` — controllers
  never call Spatie directly. (Media is the standing exception to "no Service by default": one
  place for upload/replace/delete.)
  ```php
  class MediaService
  {
      // Accepts one file, an array of files, or null (nothing uploaded).
      public function upload(Model $model, UploadedFile|array|null $files, string $collection): void
      {
          foreach (Arr::wrap($files) as $file) {
              $model->addMedia($file)->toMediaCollection($collection);
          }
      }

      // Replaces the collection only when new files were actually sent.
      public function update(Model $model, UploadedFile|array|null $files, string $collection): void
      {
          if (empty($files)) {
              return;
          }
          $model->clearMediaCollection($collection);
          $this->upload($model, $files, $collection);
      }

      public function deleteMedia(Model $model, string $collection): void
      {
          $model->clearMediaCollection($collection);
      }

      public function deleteMediaItem(Model $model, int $mediaId): void
      {
          $model->media()->whereKey($mediaId)->firstOrFail()->delete();
      }
  }
  ```
- If the project already has a different upload approach, follow it.
- Validate type/size: `image|mimes:jpg,jpeg,png,webp|max:5120`. Never trust client filenames;
  never expose private files without authorization.

### Image optimization — when the project shows user-uploaded images

Products, galleries, avatars, banners: keep quality, shrink the size. Do it with media
conversions **in the model** — the Service and controllers don't change. Skip it for PDFs and
documents.

```php
use Spatie\Image\Enums\Fit;
use Spatie\MediaLibrary\MediaCollections\Models\Media;

public function registerMediaConversions(?Media $media = null): void
{
    $this->addMediaConversion('web')
        ->fit(Fit::Max, 1600, 1600)   // shrink only, never upscale
        ->format('webp')
        ->quality(80)
        ->nonQueued();                // queued only if the project runs a queue worker

    $this->addMediaConversion('thumb')
        ->fit(Fit::Max, 400, 400)
        ->format('webp')
        ->quality(75)
        ->nonQueued();
}
```

- In Blade: `$product->getFirstMediaUrl('cover', 'web')`, the `thumb` in lists/tables; a gallery:
  `$product->getMedia('gallery')->map->getUrl('web')`.
- Needs the GD or Imagick PHP extension on the server. Extra optimizer binaries (jpegoptim,
  pngquant…) are used automatically if present — not required.

## Data & performance

- Eager-load relations (`with()`), and turn lazy loading into an error during development:
  ```php
  // AppServiceProvider::boot()
  Model::preventLazyLoading(! app()->isProduction());
  ```
- `withCount('products')` / `withSum(...)` instead of loading relations just to count them;
  `with('category:id,name')` when only a few columns are needed (keep the id).
- Paginate lists that can grow; keep search/filter server-side via query params and preserve them
  across pages with `->withQueryString()`.
- `DB::transaction()` for multi-write consistency (order + items, stock + movement); with
  `lockForUpdate()` on the row being decremented (stock, seats, balance). Not for single-record CRUD.
- Huge datasets: `chunkById()` / `lazyById()`, never `->get()` on hundreds of thousands of rows.
- No caching/Redis/queues without a real requirement. If you cache, `Cache::forget()` the key when
  the data changes.

## Config & environment

- `env()` only inside `config/*.php`; everywhere else `config('services.x.key')` — after
  `config:cache`, `env()` returns null in production.
- `APP_DEBUG=false` in production. Debugbar/Telescope only as `--dev` / behind admin auth.

## RTL & localization

- Follow the project's existing localization; use `lang/` files for repeated UI text.
- Single-language project → no localization package.
- Bilingual: if the project already has a session-based `LangMiddleware` + a
  `change-language/{lang}` route (with an allow-list of locales), follow it. A new bilingual project
  → `mcamara/laravel-localization`, with every web route inside its localized group.
- Locale `ar`: `dir="rtl"` on `<html>` and Bootstrap's RTL build (`bootstrap.rtl.min.css`).
  Never hardcode `left`/`right` spacing — use Bootstrap's `ms-`/`me-`/`ps-`/`pe-` utilities.

## Accessibility & UX

Semantic HTML, a `<label>` linked to every input, `alt` on images, `aria-label` on icon-only
buttons, visible focus states, a loading state for every submit, and a clear message for every
failure.

## Official docs — when something isn't covered here

If the task needs something this skill doesn't cover (mail, notifications, queues, scheduling,
broadcasting, a helper you're unsure of), **read the official page before writing code** instead of
relying on memory. Match the docs version to `laravel/framework` in `composer.json`
(e.g. `^12` → `12.x`):

- Laravel: `https://laravel.com/docs/{version}` — e.g. `/validation`, `/eloquent-relationships`,
  `/authorization`, `/filesystem`, `/blade`, `/http-tests`, `/rate-limiting`, `/localization`.
- Spatie MediaLibrary: `https://spatie.be/docs/laravel-medialibrary`.
- laravel/ui: `https://github.com/laravel/ui`.

The official docs win on **how an API works**; this skill wins on **style decisions** (Blade-only,
FormRequests, controller-first, Toastr, the loader).

## Troubleshooting

Common errors and their fixes (419, `Add [x] to fillable`, 1451/1452, lazy load, images 404,
`env()` null on the server, permissions…): see `references/troubleshooting.md`.
