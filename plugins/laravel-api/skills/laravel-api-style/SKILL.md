---
name: laravel-api-style
description: Conventions for building Laravel REST APIs — logic in controllers, FormRequests, one fixed JSON response envelope, API Resources, Sanctum Bearer tokens, role middleware. Use whenever building or extending a Laravel API for a mobile app, a SPA, or external clients: starting a new API project, adding an endpoint, writing a Controller / FormRequest / API Resource / model, wiring Sanctum auth or role middleware, scaffolding CRUD, bookings/orders with stock or seats — even if the skill isn't named. Follow it over generic "best practices" when they differ. Trigger words: Laravel API, REST API, new endpoint, controller, FormRequest, API Resource, Sanctum, token, CRUD endpoint, لارافيل API. Not for Blade-only (server-rendered) projects — use laravel-blade-style there.
---

# Laravel API style

When writing Laravel API code, match **this** — even where it differs from textbook "best
practice". The goal is code that is clean, easy to maintain, and reads like one person wrote it.
Defaults are for ordinary CRUD; see **Scaling up** for when to add more structure.

## Before writing code

1. Read the project's `CLAUDE.md` (if any). Otherwise confirm what matters: entities, roles,
   whether OTP / media / a Blade side / localization apply, and the key business rules.
2. Inspect existing routes, controllers, requests, resources, and models — reuse their patterns.
3. Check `composer.json` before suggesting a package.
4. For a multi-step feature, state a short plan first.

## Baseline rules (always)

- **Security:** validate every input server-side; authorize on the server; never trust a
  client-sent `user_id`/owner id — derive it from `$request->user()`; never return passwords,
  OTP codes, other users' tokens, stack traces, or SQL errors; never commit `.env` or `vendor/`.
- **Tests:** feature tests per endpoint — happy path, validation failure (422), and
  auth/ownership failure (401/403) when one exists.
- **Change discipline:** don't install packages, restructure folders, or change an existing
  endpoint's response shape without checking in first — clients depend on it.
- **Comments are opt-in:** only for a non-obvious *why* (a lock, a transaction, a non-standard
  status code).

## Core defaults

- **PHP 8.2+, Laravel 11/12+, MySQL + Eloquent. API-only, JSON.** No Blade unless the project has a
  web side (see `references/opt-in-modules.md`).
- **Controllers hold the logic, by default** — queries, `DB::transaction`, `lockForUpdate`, guards.
  Move logic into a Service only when Scaling up says so. Media is the standing exception
  (`MediaService`, see opt-in).
- **Domain-grouped controllers:** `App\Http\Controllers\Api\{Domain}\{Domain}Controller`.
- **Validation is always a FormRequest** — never `Validator::make()` or `$request->validate()` in a
  controller. All requests extend a small `ApiFormRequest` base so a 422 keeps the envelope.
- **One response envelope, everywhere:** boolean `status`, a `message`, the payload under `data`.
  The single most important house rule.
- **API Resources always shape output.** Never return a raw model. Eager-load with `->with()` in
  the controller; expose relations with `whenLoaded()`.
- **Auth is Sanctum.** `createToken` to issue, `auth:sanctum` to protect, `role:...` middleware for
  role gates, `currentAccessToken()->delete()` to log out.
- **Errors are guarded per method** — guard first, return early. The only global piece: every
  `api/*` error renders as JSON (below).
- **Status codes:** `200` ok · `201` created · `400` bad request · `401` unauthenticated ·
  `403` forbidden · `404` not found · `409` conflict (e.g. no seats left) · `422` validation ·
  `429` too many requests. Keep them accurate.
- **RESTful URIs:** `Route::apiResource('categories', CategoryController::class)` →
  `GET /categories`, `POST /categories`, `GET /categories/{category}`, … — never dotted URIs like
  `/category.create` or verbs in the path.
- **Models:** `$fillable` whitelist, `$hidden` for secrets, `casts()` for dates / `hashed` /
  booleans / enums, relationships on both sides. Foreign keys `restrictOnDelete()` by default,
  `cascadeOnDelete()` only for children meaningless without the parent.

## The response envelope

```php
// success — single
return response()->json([
    'status'  => true,
    'message' => 'Event created successfully',
    'data'    => new EventResource($event),
], 201);

// success — collection
'data' => EventResource::collection($events),

// success — paginated list
'data' => EventResource::collection($events),   // $events = Event::paginate(15)
'meta' => [
    'current_page' => $events->currentPage(),
    'last_page'    => $events->lastPage(),
    'per_page'     => $events->perPage(),
    'total'        => $events->total(),
],

// login / register
'data' => ['token' => $token, 'user' => new UserResource($user)],

// error
return response()->json([
    'status'  => false,
    'message' => 'Event not found',
], 404);

// validation (produced automatically by ApiFormRequest)
['status' => false, 'message' => 'Validation error', 'errors' => [...]]
```

- No `success` key — `status` is the only flag. `errors` appears only on `422`.
- Delete returns `200` with a message (keeps the envelope), not an empty `204`.

## Canonical slice (copy this shape for a new endpoint)

```php
// app/Http/Requests/ApiFormRequest.php — base request; keeps the envelope on 422
abstract class ApiFormRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    protected function failedValidation(Validator $validator): void
    {
        throw new HttpResponseException(response()->json([
            'status'  => false,
            'message' => 'Validation error',
            'errors'  => $validator->errors(),
        ], 422));
    }
}

// app/Http/Requests/StoreEventRequest.php
class StoreEventRequest extends ApiFormRequest
{
    public function rules(): array
    {
        return [
            'title'       => 'required|string|max:255',
            'start_time'  => 'required|date|after:now',
            'seats'       => 'required|integer|min:1',
            'category_id' => 'required|integer|exists:categories,id',
        ];
    }
}

// app/Http/Resources/EventResource.php
class EventResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'       => $this->id,
            'title'    => $this->title,
            'category' => new CategoryResource($this->whenLoaded('category')),
        ];
    }
}

// app/Http/Controllers/Api/Event/EventController.php
class EventController extends Controller
{
    public function store(StoreEventRequest $request)
    {
        $event = Event::create($request->validated());

        return response()->json([
            'status'  => true,
            'message' => 'Event created successfully',
            'data'    => new EventResource($event->load('category')),
        ], 201);
    }
}
```

### Multi-step writes with stock / seats / balances

Wrap them in a transaction and lock the row you decrement, so two requests can't take the same seat:

```php
public function store(StoreBookingRequest $request, Event $event)
{
    $booking = DB::transaction(function () use ($request, $event) {
        // Lock the event row so concurrent bookings can't oversell.
        $event = Event::whereKey($event->id)->lockForUpdate()->first();

        if ($event->available_seats < $request->seats) {
            return null;
        }

        $event->decrement('available_seats', $request->seats);

        return $request->user()->bookings()->create([
            'event_id' => $event->id,
            'seats'    => $request->seats,
        ]);
    });

    if (! $booking) {
        return response()->json(['status' => false, 'message' => 'Not enough seats left'], 409);
    }

    return response()->json([
        'status'  => true,
        'message' => 'Booking created successfully',
        'data'    => new BookingResource($booking->load('event')),
    ], 201);
}
```

## Ownership (IDOR) — every user-owned record

Changing `/bookings/15` to `/bookings/16` must not show someone else's booking. Load user-owned
records **through the user**, or check a Policy:

```php
$booking = $request->user()->bookings()->with('event')->findOrFail($id);
```

Admin endpoints that see everything sit behind `role:admin`.

## JSON for every API error

So an unauthenticated or missing-route request never redirects to a login page:

```php
// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->shouldRenderJsonWhen(fn (Request $request) => $request->is('api/*'));
})
```

## Validation details

- Unique on update: `Rule::unique('users', 'email')->ignore($this->user()->id)` (or the bound model).
- `prepareForValidation()` to normalize input (trim, lowercase email, build a slug).
- Passwords: `Password::min(8)` and the `hashed` cast on the model.
- Arrays: `'items' => 'required|array|min:1'`, `'items.*.product_id' => 'required|exists:products,id'`.

## Auth & rate limiting

```php
// AppServiceProvider::boot()
RateLimiter::for('login', fn (Request $request) =>
    Limit::perMinute(5)->by(strtolower((string) $request->input('email')).'|'.$request->ip()));

// routes/api.php
Route::post('login', [AuthController::class, 'login'])->middleware('throttle:login');
```

Apply the default `api` limiter to every API route with `$middleware->throttleApi()` in
`bootstrap/app.php`. Token expiry: set `expiration` in `config/sanctum.php` when the project needs it.

## Performance

- Eager-load (`with`, `load`, `loadMissing`); count with `withCount()` / `withSum()`; select
  columns with `with('category:id,name')`.
- Catch N+1 during development:
  ```php
  Model::preventLazyLoading(! app()->isProduction());   // AppServiceProvider::boot()
  ```
- Paginate lists that can grow; `chunkById()` for large batch work.

## Config & environment

- `env()` only inside `config/*.php`; in code use `config('services.x.key')` — after
  `config:cache`, `env()` returns null on the server.
- CORS: list the frontend domain(s) in `config/cors.php`, never `*` in production.
- `APP_DEBUG=false` in production. Run `composer audit` before launch.

## Naming

- Controller methods: `index`, `show`, `store`, `update`, `destroy`.
- Requests: `Store{Model}Request`, `Update{Model}Request`. Resources: `{Model}Resource`.
- Services (when justified): singular — `MediaService`, `BookingService`.
- Web named routes (only with a Blade side): `admin.<resource>.<action>`.

## Opt-in modules — do NOT scaffold by default

Add only when the project actually has the feature; read `references/opt-in-modules.md`:

- **Media / image uploads** — Spatie MediaLibrary behind `MediaService`, with webp conversions.
- **OTP email verification** — 6-digit code flow on register/login.
- **Blade / web dual-track** — a Blade side next to the API (follow `laravel-blade-style`).
- **Localization** — `LangMiddleware`, `ar`/`en`.

## Scaling up — only when justified

Default stays simple: controller logic, no Repository, no global handler beyond JSON rendering.
Add one of these when the justification clearly applies (borderline → ask), and say why in one line:

- **Service:** logic is complex, reused across controllers/commands/jobs, or transactional across
  multiple models. Never a pass-through Service that only forwards to Eloquent.
- **Repository:** rare — only when the data source itself must be swappable in a large,
  long-lived project.
- **Policies/Gates:** per-record authorization or more than one condition. A flat `role:admin`
  middleware is enough for "admins only".
- **Business exception with `render()`:** an expected domain failure thrown from deep inside a
  Service (e.g. `InsufficientStockException`). Its `render()` must return the **same envelope**
  (`status: false`, `message`, optional `errors`), and `report()` returns `true` so it doesn't fill
  the log.
- **Global exception mapping:** per-method guards start producing inconsistent error shapes.
- **Pagination:** the list can grow unbounded.
- **API versioning (`/v1`):** external consumers that must not break.
- **Events/Jobs/Queues:** real async work (mail, SMS, heavy processing) — not CRUD.
- **Caching/Redis:** a proven, repeated, expensive read; `Cache::forget()` on change.
- **Broadcasting:** an actual real-time requirement.

## Starting a new project

1. Confirm entities, roles, opt-in modules, and key business rules.
2. `php artisan install:api` (Sanctum + `routes/api.php`), then create `ApiFormRequest`, the JSON
   rendering for `api/*`, and the `login` rate limiter.
3. Build each resource as the canonical slice; keep the envelope identical everywhere.
4. Ship a Postman collection (or OpenAPI docs) in the repo, plus feature tests.

## Official docs — when something isn't covered here

If the task needs something this skill doesn't cover (mail, notifications, queues, scheduling,
broadcasting, Passport, a helper you're unsure of), **read the official page before writing code**
instead of relying on memory. Match the version to `laravel/framework` in `composer.json`
(e.g. `^12` → `12.x`):

- Laravel: `https://laravel.com/docs/{version}` — e.g. `/sanctum`, `/eloquent-resources`,
  `/validation`, `/authorization`, `/database#database-transactions`, `/rate-limiting`,
  `/http-tests`, `/errors`.
- Spatie MediaLibrary: `https://spatie.be/docs/laravel-medialibrary`.

The official docs win on **how an API works**; this skill wins on **style decisions** (the envelope,
FormRequests, controller-first, Resources).

## Troubleshooting

Common errors and fixes (`Route [login] not defined`, `Unauthenticated.`, 1451/1452, lazy load,
CORS, `env()` null on the server…): see `references/troubleshooting.md`.
