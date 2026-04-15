---
name: snowflake-laravel
description: Snowflake data warehouse integration in Laravel using a custom PDO driver. Activate when working on service-datahub, Snowflake queries, data export jobs, or SnowflakeService.
metadata:
  tags: snowflake, laravel, pdo, data-warehouse, exports
---

## Overview

`service-datahub` connects to Snowflake using a custom PDO driver (not a standard Laravel database connection). Queries return associative arrays. Data is exported to AWS S3 via queued jobs.

---

## Connection

```php
// App\Http\Services\SnowflakeService
class SnowflakeService
{
    private \PDO $connection;

    public function __construct()
    {
        $account  = config('snowflake.account');
        $user     = config('snowflake.user');
        $password = config('snowflake.password');

        $this->connection = new \PDO(
            "snowflake:account={$account};region=eu-west-1",
            $user,
            $password
        );
    }

    public function query(string $sql): array
    {
        $result = $this->connection->query($sql);
        return $result->fetchAll(\PDO::FETCH_ASSOC) ?? [];
    }
}
```

**Important:** The DSN format is `snowflake:account={account};region=eu-west-1` — not a standard MySQL/Postgres DSN. Region is always `eu-west-1`.

---

## Config

`config/snowflake.php`:
```php
return [
    'account'  => env('SNOWFLAKE_ACCOUNT'),
    'user'     => env('SNOWFLAKE_USER'),
    'password' => env('SNOWFLAKE_PASSWORD'),
];
```

Never use `env()` directly in code — always go through `config('snowflake.*')`.

---

## Export Architecture

```
SnowflakeExportCommand (Artisan)
    └── dispatches SnowflakeExportJob
            └── dispatches SnowflakeExportTableJob (per table)
                    └── writes result to S3 via Flysystem
```

### Export Job pattern
```php
class SnowflakeExportTableJob implements ShouldQueue
{
    public function __construct(
        private string $table,
        private string $query
    ) {}

    public function handle(SnowflakeService $snowflake): void
    {
        $rows = $snowflake->query($this->query);

        Storage::disk('s3')->put(
            "exports/{$this->table}/" . now()->format('Y-m-d') . '.json',
            json_encode($rows)
        );
    }
}
```

---

## Controllers

Controllers are namespaced under `App\Http\Controllers\Snowflake\`:
- `NewsletterEventController` — newsletter engagement data
- `ActiveCorporateUsersController` — active user exports

Follow this namespace convention for new Snowflake-related controllers.

---

## Key Rules

- Snowflake uses its own PDO driver — do not use Eloquent or Query Builder
- DSN always includes `region=eu-west-1`
- Results are always `fetchAll(PDO::FETCH_ASSOC)` — no model mapping
- Export jobs write to S3 via `Storage::disk('s3')` using `league/flysystem-aws-s3-v3`
- Use queued jobs for all exports — Snowflake queries can be slow
- Config lives in `config/snowflake.php`, never call `env()` in service classes
