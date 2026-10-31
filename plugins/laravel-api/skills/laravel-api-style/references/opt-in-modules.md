# Opt-in modules

Add these **only when the project actually has the feature** — don't scaffold them by default.
Check the project's `CLAUDE.md` (if any) for which ones are enabled.

## Media / image uploads

`spatie/laravel-medialibrary`, wrapped in a service so controllers never call Spatie directly.

```php
// app/Services/MediaService.php
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

- The model implements `Spatie\MediaLibrary\HasMedia` and `use InteractsWithMedia`.
- Named collections, e.g. `event_cover` (single: `->singleFile()` in `registerMediaCollections()`)
  and `event_gallery` (multiple).
- The controller injects the service:
  ```php
  public function __construct(protected MediaService $media) {}

  $this->media->upload($event, $request->file('image'), 'event_cover');
  ```
- Validation: `'image' => 'nullable|image|mimes:jpg,jpeg,png,webp|max:5120'`,
  `'images.*' => 'image|mimes:jpg,jpeg,png,webp|max:5120'`.

### Image optimization (when the API serves user-uploaded images)

Keep quality, shrink the size — with conversions **in the model**; the service and controllers
don't change. Skip it for PDFs/documents.

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

The Resource exposes URLs, never raw media models:

```php
'image'  => $this->getFirstMediaUrl('event_cover', 'web') ?: null,
'thumb'  => $this->getFirstMediaUrl('event_cover', 'thumb') ?: null,
'images' => $this->getMedia('event_gallery')->map->getUrl('web'),
```

Needs GD or Imagick on the server. Optimizer binaries (jpegoptim, pngquant…) are used if present.

## OTP email verification

- On register: generate a 6-digit `otp` (`random_int(100000, 999999)`), store `otp`,
  `otp_expires_at` (`now()->addMinutes(10)`), `is_otp_verified = false`; mail the code; issue the
  Sanctum token.
- `resendOtp`: regenerate + re-mail, throttled (`throttle:2,1`).
- `verifyEmailOtp` (FormRequest): reject if wrong or expired (`422`); "already verified" can be
  `200`. On success set `is_otp_verified = true`, null `otp` / `otp_expires_at`, return a fresh token.
- `login`: block with `403` until `is_otp_verified` is true.
- Add the fields to `$fillable`, `otp` to `$hidden`, cast `otp_expires_at` to `datetime`.
- Never return the OTP in any response — it only goes out by mail.

## Blade / web dual-track

When the same app also serves server-rendered pages (e.g. an admin panel next to the API):

- Web routes in `routes/web.php`, API routes in `routes/api.php` — never mixed.
- Web controllers live outside `Api/` and return views/redirects; API controllers keep the envelope.
- Logic both sides need goes into a Service (the "reused across controllers" case) — not duplicated.
- Follow the `laravel-blade-style` skill (the `laravel-blade` plugin) for the web side, if installed.

## Localization

- `LangMiddleware` reads an `Accept-Language` / `lang` header (or query param), checks it against
  an allow-list (`['ar', 'en']`), and calls `app()->setLocale(...)`; default per project.
- Messages via `__('...')` with `lang/{ar,en}` files when the project is bilingual — check whether
  they already exist before assuming scope.
