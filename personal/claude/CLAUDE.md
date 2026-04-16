# Personal Development Context

## Primary Stack

### Backend
- **PHP / Laravel** — primary language and framework
- **Lumen** — legacy microservices, being phased out; avoid introducing new Lumen code where possible
- **Python** — minimal usage, treat as secondary

### Frontend
- **Vue 3** — preferred for new frontend work
- **Vue 2** — existing codebases; do not silently upgrade patterns to Vue 3
- **Blade** — Laravel templating, used alongside Vue

---

## Docker — Read This First

All projects run inside Docker containers. **Never run `php`, `artisan`, `composer`, `npm`, `./vendor/bin/*` directly on the host.** Use `make` targets instead. If no make target exists for what you need, tell the user to run it inside the container — do not attempt raw `docker compose exec` commands.

Key examples:
- `make test` or `make inline-test` — not `php artisan test`
- `make pint` or `make pint-fix` — not `./vendor/bin/pint`
- `make install` — not `composer install && npm install`

Always check the project's `Makefile` before running any command.

---

## General Preferences

- Follow Laravel conventions: Eloquent ORM, service providers, artisan commands, form requests, policies
- Prefer Laravel built-ins over third-party packages when the built-in is sufficient
- Use Laravel's validation (Form Requests) rather than inline validation in controllers
- Respect existing patterns in each codebase before suggesting refactors
- When touching Lumen code, prefer minimal changes — flag opportunities to migrate to Laravel if relevant

## PHP Style
- Follow Spatie PHP/Laravel guidelines (loaded via `php-guidelines-from-spatie` skill)
- PHPUnit for tests (not Pest, unless the project already uses it)

## Vue Style
- Vue 3: Composition API with `<script setup>` for new components
- Vue 2: Options API — do not mix in Composition API unless the project already uses `@vue/composition-api`
- Keep components focused; prefer composables (Vue 3) or mixins (Vue 2) for shared logic

## Build Tooling
- **Vite** — newer projects (family_tree, etc.)
- **Laravel Mix v6** (Webpack) — older projects (frontend-borsen-2021, etc.)
- Check `package.json` or project root for `vite.config.js` vs `webpack.mix.js` before suggesting build commands

## Python Style
- PEP 8
- Keep it simple — these are typically small scripts or utilities

---

## What to Avoid
- Do not suggest upgrading Vue 2 to Vue 3 unless explicitly asked
- Do not suggest migrating Lumen to Laravel unless the topic comes up
- Do not introduce new dependencies without flagging them
- Do not run CLI tools directly on the host — always use `make` targets
