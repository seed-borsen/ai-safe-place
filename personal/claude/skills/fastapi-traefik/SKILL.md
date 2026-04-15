---
name: fastapi-traefik
description: FastAPI application development with Traefik reverse proxy integration. Activate when working on devhub, develop-proxy, dynamic routing config, Traefik middleware, or the FastAPI control panel.
metadata:
  tags: fastapi, traefik, python, uvicorn, sqlite, async
---

## Overview

`devhub` is a FastAPI-based control panel for managing local development services. It integrates with Traefik v3 for dynamic routing — the panel writes Traefik config files that Traefik watches and reloads automatically.

**Stack:**
- `fastapi==0.115.6` + `uvicorn[standard]==0.34.0`
- `aiosqlite==0.20.0` — async SQLite
- `jinja2==3.1.5` — HTML templates
- `httpx==0.28.1` — async HTTP client
- `docker==7.1.0` — container management
- `pyyaml==6.0.2` — Traefik config generation

---

## Application Structure

```
panel/app/
├── main.py           — FastAPI app, lifespan, router mounting
├── config.py         — settings (PATH_PREFIX, DATABASE_PATH, etc.)
├── database.py       — init_db, get_db, get_enabled_services
├── config_generator.py — Traefik YAML config generation
├── health_checker.py — background health check task
└── routes/
    ├── api.py        — JSON API routes
    └── html.py       — Jinja2 HTML routes
```

---

## App Initialization (Lifespan)

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from .config import settings
from .database import init_db, get_db, get_enabled_services
from .config_generator import regenerate_all_configs, generate_system_configs
from .health_checker import run_health_checker
import asyncio

@asynccontextmanager
async def lifespan(app: FastAPI):
    await init_db()
    generate_system_configs()
    db = await get_db()
    services = await get_enabled_services(db)
    regenerate_all_configs(services)
    health_checker_task = asyncio.create_task(run_health_checker())
    yield
    health_checker_task.cancel()

app = FastAPI(lifespan=lifespan, root_path=settings.PATH_PREFIX)
```

Always use `lifespan` context manager — not deprecated `on_event` handlers.

---

## Traefik Path-Stripping Integration

Traefik strips the `/proxy` prefix before forwarding to FastAPI. The app must be mounted at the root but aware of the prefix for URL generation.

**Traefik config (`traefik/traefik.yml`):**
```yaml
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
  websecure:
    address: ":443"

providers:
  file:
    directory: /etc/traefik/dynamic/
    watch: true
```

**Dynamic route config (generated YAML):**
```yaml
http:
  routers:
    devhub:
      rule: "Host(`seed.borops.net`) && PathPrefix(`/proxy`)"
      middlewares:
        - strip-proxy
      service: devhub
  middlewares:
    strip-proxy:
      stripPrefix:
        prefixes:
          - "/proxy"
  services:
    devhub:
      loadBalancer:
        servers:
          - url: "http://panel:8000"
```

**In FastAPI, set template base path globally:**
```python
from fastapi.templating import Jinja2Templates

templates = Jinja2Templates(directory="app/templates")
templates.env.globals["base_path"] = settings.PATH_PREFIX
```

Use `base_path` in all template hrefs so links work behind the prefix.

---

## Async SQLite Pattern

```python
import aiosqlite

async def init_db():
    async with aiosqlite.connect(settings.DATABASE_PATH) as db:
        await db.execute("""
            CREATE TABLE IF NOT EXISTS services (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                enabled INTEGER DEFAULT 1
            )
        """)
        await db.commit()

async def get_enabled_services(db: aiosqlite.Connection) -> list[dict]:
    async with db.execute("SELECT * FROM services WHERE enabled = 1") as cursor:
        rows = await cursor.fetchall()
        return [dict(row) for row in rows]
```

---

## Config Regeneration

When services are added/updated/removed, regenerate Traefik configs and write to the watched directory:

```python
import yaml
from pathlib import Path

def regenerate_all_configs(services: list[dict]) -> None:
    config_dir = Path("/etc/traefik/dynamic/")

    for service in services:
        config = build_traefik_config(service)
        config_path = config_dir / f"{service['name']}.yml"
        config_path.write_text(yaml.dump(config))
```

Traefik watches the directory and hot-reloads — no restart needed.

---

## Static Files

```python
from fastapi.staticfiles import StaticFiles

app.mount("/static", StaticFiles(directory="/app/static"), name="static")
```

Reference in templates: `{{ base_path }}/static/style.css`

---

## Key Rules

- Always use `lifespan` for startup/shutdown — not `@app.on_event`
- Traefik strips the path prefix — use `root_path=settings.PATH_PREFIX` on the FastAPI app and `base_path` in templates
- Config files are written to `/etc/traefik/dynamic/` — Traefik hot-reloads them
- Use `aiosqlite` for all database access — this is an async application
- Health checker runs as a background `asyncio.Task` — cancel it on shutdown
- `generate_system_configs()` is synchronous (file I/O only); `regenerate_all_configs()` takes the service list from the DB
