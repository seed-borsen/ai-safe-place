---
name: statamic-cms
description: Statamic CMS integration and custom element architecture in frontend-borsen-2021. Activate when working on Statamic content types, custom Blade elements, the multi-module Laravel architecture, or GraphQL integration in this project.
metadata:
  tags: statamic, laravel, vue2, blade, multi-module, graphql
---

## Overview

`frontend-borsen-2021` is a Laravel 12 application with a multi-module architecture and Vue 2 components rendered via Blade "elements". It uses `firebase/php-jwt` for token-based auth, `jenssegers/agent` for user-agent detection, and `spatie/laravel-responsecache` for HTTP response caching.

**Frontend stack:** Vue 2.6 + Bootstrap Vue + Laravel Mix v6 (Webpack, not Vite)

---

## Multi-Module Architecture

Multiple PSR-4 namespaces registered in `composer.json`:

```json
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Borsen\\SDK\\": "modules/sdk/src/",
        "Brapi\\": "modules/brapi/app/",
        "Datalayer\\": "modules/Datalayer/app/",
        "EventBuizz\\": "modules/EventBuizz/app/",
        "DynamicPaywall\\": "modules/DynamicPaywall/app/",
        "Paywall\\": "modules/Paywall/app/",
        "Uxi\\": "modules/Uxi/app/",
        "Customer\\": "modules/Customer/app/"
    }
}
```

Each module has its own `app/` directory. Service providers, routes, and views are registered per module. New modules follow the same pattern — add namespace to `composer.json` and create a service provider.

---

## Custom Blade Elements

Vue components are embedded in Blade via custom "element" files:

**Location:** `app/Http/Elements/{ComponentName}/{component-name}-element.blade.php`

**Examples:**
- `RelatedArticlesList/related-articles-list-element.blade.php`
- `ArticlePicture/article-picture-element.blade.php`
- `ArticleTitle/article-title-element.blade.php`
- `RightNowTicker/right-now-ticker-element.blade.php`
- `EventBuizzSpeakerList/eventbuizz-speaker-list-element.blade.php`

**Pattern — element blade file:**
```blade
<related-articles-list
    :article-id="{{ $articleId }}"
    :limit="{{ $limit ?? 5 }}"
></related-articles-list>
```

**Usage in templates:**
```blade
@include('elements.related-articles-list.related-articles-list-element', [
    'articleId' => $article->id,
])
```

Follow PascalCase for the directory/component name, kebab-case for the blade filename and Vue component tag.

---

## Vue 2 Component Conventions

This project uses Vue 2 (Options API). Do **not** use Vue 3 or Composition API patterns here.

```javascript
// resources/js/components/RelatedArticlesList.vue
export default {
    name: 'RelatedArticlesList',
    props: {
        articleId: {
            type: Number,
            required: true,
        },
        limit: {
            type: Number,
            default: 5,
        },
    },
    data() {
        return {
            articles: [],
        };
    },
    mounted() {
        this.fetchArticles();
    },
    methods: {
        async fetchArticles() {
            // ...
        },
    },
};
```

Register components globally in `resources/js/app.js` so they're available across all Blade templates.

---

## Build Tooling

Uses **Laravel Mix v6** (not Vite):

```javascript
// webpack.mix.js
const mix = require('laravel-mix');

mix.js('resources/js/app.js', 'public/js')
   .vue({ version: 2 })
   .sass('resources/sass/app.scss', 'public/css');
```

**Commands:**
```bash
npm run dev      # development build (watch)
npm run prod     # production build
npm run hot      # hot module reloading
```

Do not suggest migrating to Vite unless explicitly asked.

---

## GraphQL Integration

The project integrates with a GraphQL API for content. Use `guzzlehttp/guzzle` or Laravel's HTTP client for queries:

```php
$response = Http::post(config('services.graphql.endpoint'), [
    'query' => '{ articles(limit: 10) { id title } }',
]);

$articles = $response->json('data.articles');
```

---

## JWT Token Authentication

Uses `firebase/php-jwt` for token generation/validation (not Laravel Sanctum or Passport):

```php
use Firebase\JWT\JWT;
use Firebase\JWT\Key;

$token = JWT::encode(['user_id' => $user->id, 'exp' => time() + 3600], config('app.key'), 'HS256');
$decoded = JWT::decode($token, new Key(config('app.key'), 'HS256'));
```

---

## Response Caching

`spatie/laravel-responsecache` is active — be aware that controller responses may be cached.

Add `DoNotCacheResponse` to routes/controllers that must always be fresh:

```php
use Spatie\ResponseCache\Middlewares\DoNotCacheResponse;

Route::get('/live-ticker', LiveTickerController::class)
    ->middleware(DoNotCacheResponse::class);
```

Clear cache when needed: `php artisan responsecache:clear`

---

## Borsen SDK

The SDK is an internal package in `modules/sdk/`:
- Published via `php artisan borsen-sdk:publish:theme` (runs post-autoload)
- Contains shared theme assets, base classes, and shared blade components

Do not bypass the SDK publish step — theme assets depend on it.

---

## Key Rules

- Vue 2 only — Options API, no Composition API, no `<script setup>`
- Laravel Mix v6 (Webpack) — do not suggest Vite
- New Vue components = new element blade file in `app/Http/Elements/`
- Each module is a separate PSR-4 namespace — add new modules to `composer.json`
- Response caching is active — mark dynamic routes with `DoNotCacheResponse`
