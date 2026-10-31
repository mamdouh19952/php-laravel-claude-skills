<div dir="rtl">

# سكيلات PHP و Laravel لـ Claude Code

مجموعة سكيلات (Skills) بتخلّي **Claude Code** يكتب كود PHP بأسلوب واحد: **بسيط، نضيف، آمن، وسهل
الصيانة**، ومن غير over-engineering.

| البلجن | بيستخدم في |
|---|---|
| **laravel-blade** | مشاريع Laravel بتعرض صفحات Blade: لوحات تحكم، متاجر، أنظمة صغيرة. Bootstrap و vanilla JS و Toastr |
| **laravel-api** | Laravel REST API لموبايل أو SPA: شكل JSON واحد، و FormRequests، و Resources، و Sanctum |
| **native-php** | مواقع PHP + MySQL بسيطة من غير framework: PDO، و CSRF، و sessions، ورفع صور آمن |
| **shared-hosting-deploy** | رفع Laravel أو PHP عادي على استضافة مشتركة (cPanel / Hostinger hPanel) من غير ما `.env` يتكشف |

---

## التسطيب

جوه Claude Code:

```
/plugin marketplace add mamdouh19952/php-laravel-claude-skills
```

وبعدين سطّب اللي محتاجه بس:

```
/plugin install laravel-blade@php-laravel-claude-skills
/plugin install laravel-api@php-laravel-claude-skills
/plugin install native-php@php-laravel-claude-skills
/plugin install shared-hosting-deploy@php-laravel-claude-skills
```

السكيل بيشتغل لوحده لما تطلب حاجة تخصه، ومش محتاج تنادي عليه بالاسم.

## أمثلة طلبات

- `اعمل CRUD للمنتجات بالصور في لوحة الأدمن` ← **laravel-blade**
- `ضيف endpoint لحجز مقاعد في الإيفنت` ← **laravel-api**: transaction و lock عشان الكرسي مايتحجزش مرتين
- `اعمل موقع مهام بسيط PHP و MySQL فيه login` ← **native-php**
- `ارفع المشروع على cPanel تحت example.com/app` ← **shared-hosting-deploy**

## الأفكار اللي ماشيين عليها

- **الأبسط الأول:** الـ logic في الـ controller، وبنضيف Service أو Repository **لما يبقى فيه سبب حقيقي بس**، والسكيل بيقول السبب.
- **الأمان مش اختياري:**
  - validation على السيرفر.
  - CSRF في كل فورم.
  - كل يوزر يشوف بياناته بس (IDOR).
  - prepared statements في أي query.
  - ممنوع أسرار أو stack traces توصل لليوزر.
- **تحسين الصور:** لما المشروع فيه صور بيرفعها المستخدم، بتتحفظ webp بحجم صغير وجودة كويسة.
- **عربي و RTL:** دعم كامل لمشاريع عربي أو إنجليزي أو الاتنين.
- **المراجع الرسمية:** لو الطلب فيه حاجة السكيل مش مغطيها، Claude بيرجع لـ docs Laravel أو PHP الرسمية على نفس إصدار مشروعك بدل ما يعتمد على ذاكرته.

## التحديث

```
/plugin marketplace update php-laravel-claude-skills
```

## الرخصة

MIT. استخدمه وعدّل عليه براحتك.

</div>

---

# PHP & Laravel skills for Claude Code

A set of Claude Code skills that make Claude write PHP in one consistent style: **simple, clean,
secure, and easy to maintain**, with no over-engineering.

| Plugin | For |
|---|---|
| **laravel-blade** | Server-rendered Laravel apps (admin panels, stores, small business systems): Blade + Bootstrap + vanilla JS + Toastr |
| **laravel-api** | Laravel REST APIs for mobile apps / SPAs: one JSON envelope, FormRequests, API Resources, Sanctum |
| **native-php** | Simple PHP + MySQL sites without a framework: PDO, CSRF, sessions, safe image uploads |
| **shared-hosting-deploy** | Deploying Laravel or plain PHP to shared hosting (cPanel / Hostinger hPanel) without exposing `.env` |

## Install

Inside Claude Code:

```
/plugin marketplace add mamdouh19952/php-laravel-claude-skills
```

Then install only what you need:

```
/plugin install laravel-blade@php-laravel-claude-skills
/plugin install laravel-api@php-laravel-claude-skills
/plugin install native-php@php-laravel-claude-skills
/plugin install shared-hosting-deploy@php-laravel-claude-skills
```

Skills activate automatically when your request matches. You don't need to name them.

## Example prompts

- "Build a products CRUD with images in the admin panel" → **laravel-blade**
- "Add an endpoint to book seats for an event" → **laravel-api** (transaction + row lock, so a seat can't be sold twice)
- "Make a simple PHP + MySQL task app with login" → **native-php**
- "Deploy this to cPanel under example.com/app" → **shared-hosting-deploy**

## Principles

- **Simplest thing first:** logic lives in the controller. A Service or Repository is added
  **only when there's a real reason**, and the skill says why.
- **Security is not optional:**
  - server-side validation
  - CSRF on every form
  - users only see their own records (IDOR)
  - prepared statements for every query
  - no secrets or stack traces shown to users
- **Image optimization:** user-uploaded images are stored as small, good-quality webp.
- **Arabic & RTL:** full support for Arabic, English, or bilingual projects.
- **Official docs:** for anything a skill doesn't cover, Claude reads the official Laravel / PHP
  docs for *your* project's version instead of guessing.

## Update

```
/plugin marketplace update php-laravel-claude-skills
```

## License

MIT. Use it and change it freely.
