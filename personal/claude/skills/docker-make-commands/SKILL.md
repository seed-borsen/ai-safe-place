---
name: docker-make-commands
description: How to run PHP, artisan, composer, npm, and code quality tools in Docker-based projects. Activate whenever you would run php, artisan, composer, npm, pint, phpstan, rector, or phpunit in any of these repos.
metadata:
  tags: docker, make, laravel, php, artisan, composer, npm
---

## Core Rule

**Never run `php`, `artisan`, `composer`, `npm`, `./vendor/bin/*` directly from the host.** All commands must go through `make`. The application code lives inside Docker containers — host execution does not work.

---

## The `make test` Trap

In most repos, `make test` opens an **interactive bash shell** inside the container. It does NOT run `php artisan test` automatically. An agent cannot use interactive shells.

**Do not use `make test` directly.** Use the non-interactive alternatives below:

| Repo | Non-interactive test command |
|------|------------------------------|
| `service-brapi` | `make test` ✅ (runs `php artisan test` directly) |
| `service-datasync` | `make test` ✅ (runs `php artisan test` directly) |
| `frontend-norkon` | `make inline-test` ✅ |
| `frontend-borsen-2021` | No non-interactive test target — ask user |
| `service-customer` | No test target — ask user |
| `service-datahub` | No test target — ask user |
| `frontend-webmaster-2021` | No test target — ask user |

When in doubt, check the Makefile before running `make test`.

---

## Pint (Code Style)

Never run `./vendor/bin/pint` directly — it won't work from the host.

| Target | What it does |
|--------|-------------|
| `make pint` | Check style, exit 0 even on violations (`\|\| true`) |
| `make pint-dirty` | Check only git-dirty files (frontend-borsen-2021) |
| `make pint-fix` | Fix style violations (available in frontend-borsen-2021, frontend-norkon) |
| `make review` | Runs pint check + phpstan (frontend-norkon) |
| `make qa` | Runs rector + pint + stan + test (service-brapi, service-datasync) |

Not all repos have all targets — check the Makefile first.

---

## Common Make Targets (all repos)

```bash
make up              # Start containers (detached)
make up-build        # Rebuild and start containers
make down            # Stop and remove containers
make restart         # down + up
make logs            # Follow container logs
make install         # Full install: pull images, build, composer install, npm install
```

---

## Running Artisan Commands

For repos without a dedicated make target, artisan commands must be run by the user interactively inside the container:

```bash
# User enters the container:
make bash            # or make bash-webserver

# Then inside the container:
php artisan migrate
php artisan db:seed
php artisan cache:clear
php artisan queue:work
```

For repos where `make test` runs artisan inline (service-brapi, service-datasync), the pattern is:
```bash
# Makefile does: docker compose run --rm webserver bash -c "php artisan test"
make test
```

If you need to run a one-off artisan command non-interactively in a repo that supports it, check whether there's an existing make target first. If not, flag to the user that they need to run it manually inside the container.

---

## Composer

```bash
make install            # runs composer install inside container (all repos)
make composer-install   # explicit target (service-datahub — includes --ignore-platform-req=ext-curl)
make composer-update    # (service-datahub only)
```

Never run `composer install` or `composer require` from the host.

---

## NPM / Frontend Builds

```bash
make npm-dev     # npm run dev (frontend-borsen-2021, frontend-norkon)
make npm-prod    # npm run prod (frontend-borsen-2021)
make npm-watch   # npm run watch (frontend-borsen-2021, frontend-norkon)
make npm-build   # npm run build (frontend-norkon)
make bash-npm    # opens interactive node container shell
```

For cdn-global-scripts (no docker-compose, uses `docker run`):
```bash
make build       # npm install + npm run build (ephemeral container)
make watch       # npm install + npm run watch
make bash        # interactive node container
```

---

## Code Quality

```bash
make pint        # Laravel Pint code style check
make pint-fix    # Laravel Pint auto-fix
make stan        # PHPStan static analysis
make rector      # Rector refactoring/upgrades
make qa          # All of the above + tests (service-brapi, service-datasync)
make coverage    # PHPUnit coverage report (text output)
make coverage-report  # PHPUnit HTML coverage report
```

Never run `./vendor/bin/pint`, `./vendor/bin/phpstan`, or `./vendor/bin/rector` directly.

---

## Repo-Specific Quirks

### service-rabbit
- Uses `docker exec -it service-rabbit-directus bash` (not docker-compose)
- Most commands delegate to `./development/scripts/makecommands.sh`
- Available: `make up`, `make down`, `make logs`, `make bash`, `make DB_from_stage`, `make DB_from_develop`

### service-datahub
- Composer needs `--ignore-platform-req=ext-curl` — always use `make composer-install`
- Has `make show-snowflake-tables` — runs `php artisan snowflake:show-tables`

### service-brapi / service-datasync
- `make create-db` — creates databases and runs migrations (needed after fresh install)
- `make qa` — full quality check chain: rector → pint → stan → test
- `make get-data-from-iteras` — imports campaigns and customers

### frontend-webmaster-2021
- `make clearcache` — clears Statamic stache and static cache (`php please stache:clear`)
- No test target

### frontend-norkon
- `make inline-test` — the correct non-interactive test command
- `make review` — runs pint check + phpstan together

---

## When No Make Target Exists

If a task requires running a command that has no make target:
1. Tell the user what command needs to be run
2. Tell them to enter the container with `make bash` (or `make bash-webserver`)
3. Provide the exact command to run once inside

Do not attempt to construct `docker compose exec` or `docker compose run` commands manually — the user group flags and environment setup in the Makefile are required for correct behavior.
