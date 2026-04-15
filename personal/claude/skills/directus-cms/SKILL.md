---
name: directus-cms
description: Directus 10 headless CMS with Laravel API backend. Activate when working on service-rabbit, Directus extensions (displays, interfaces, layouts, modules), or the hybrid Directus+Laravel architecture.
metadata:
  tags: directus, headless-cms, laravel, node, extensions
---

## Overview

`service-rabbit` runs two separate applications sharing a MySQL database:
- **Directus 10** (Node.js) — headless CMS, content management UI and REST/GraphQL API
- **Laravel 8** (PHP) — application API layer, business logic

Neither is a plugin of the other — they are independent processes that share a database.

---

## Architecture

```
┌─────────────────────┐     ┌──────────────────────┐
│   Directus 10       │     │   Laravel 8          │
│   (Node.js :8055)   │     │   (PHP Apache)       │
│   Content UI + API  │     │   Business Logic API │
└────────┬────────────┘     └──────────┬───────────┘
         │                             │
         └──────────┬──────────────────┘
                    │
             ┌──────▼──────┐
             │   MySQL     │
             │  (shared)   │
             └─────────────┘
```

The Laravel app may read from Directus-managed tables, but schema migrations for Directus tables live in `database/directus/` and are tracked via `Borsen\Directus\Migration` namespace.

---

## Directus Extensions

Custom extensions live in `resources/directus/extensions/`:

```
extensions/
├── displays/     — custom field value displays (read-only rendering)
├── interfaces/   — custom field input components (editing UI)
├── layouts/      — custom collection view layouts
└── modules/      — custom top-level Directus modules (sidebar items)
```

### Extension Structure

Each extension is a Node.js package:

```
extensions/interfaces/my-interface/
├── package.json
├── src/
│   └── index.js      — extension entry point
└── dist/
    └── index.js      — built output
```

**Interface example (`src/index.js`):**
```javascript
import { defineInterface } from '@directus/extensions-sdk';
import InterfaceComponent from './interface.vue';

export default defineInterface({
    id: 'my-custom-interface',
    name: 'My Custom Interface',
    icon: 'box',
    description: 'A custom input interface',
    component: InterfaceComponent,
    options: [
        {
            field: 'placeholder',
            name: 'Placeholder',
            type: 'string',
            meta: {
                interface: 'input',
            },
        },
    ],
    types: ['string'],
});
```

**Display example:**
```javascript
import { defineDisplay } from '@directus/extensions-sdk';

export default defineDisplay({
    id: 'my-display',
    name: 'My Display',
    icon: 'box',
    component: DisplayComponent,
    options: [],
    types: ['string'],
});
```

---

## Directus Database Migrations

Directus schema changes are tracked in `database/directus/` with the `Borsen\Directus\Migration` namespace (autoloaded via `composer.json`).

These are PHP migration classes that document Directus collection/field changes:

```php
namespace Borsen\Directus\Migration;

class CreateArticlesCollection
{
    public function up(): void
    {
        // Document the Directus collection structure
        // Actual schema changes are applied via Directus UI or API
    }
}
```

---

## Laravel ↔ Directus Database Access

The Laravel app can query Directus-managed tables directly using the DB facade:

```php
// Reading Directus content from Laravel
$articles = DB::table('articles')
    ->where('status', 'published')
    ->orderByDesc('date_created')
    ->get();
```

Do not use Eloquent models for Directus tables unless they're also managed by Laravel migrations. Use `DB::table()` for direct access.

---

## Directus Configuration

Directus is configured via environment variables (`.env`):

```
DIRECTUS_KEY=
DIRECTUS_SECRET=
DB_CLIENT=mysql
DB_HOST=
DB_PORT=3306
DB_DATABASE=
DB_USER=
DB_PASSWORD=
ADMIN_EMAIL=
ADMIN_PASSWORD=
```

---

## Key Rules

- Directus and Laravel are **separate processes** — they do not call each other's code
- Shared state is through the **database only**
- Directus schema migrations are tracked in `database/directus/` — not standard Laravel migrations
- Extensions are Node.js packages using `@directus/extensions-sdk` — not PHP
- Use `DB::table()` (not Eloquent) when Laravel reads Directus-managed tables
- Directus extensions (interfaces, displays, etc.) must be built (`npm run build`) before they appear in the UI
- Do not modify Directus system tables (`directus_*`) directly — use the Directus API or UI
