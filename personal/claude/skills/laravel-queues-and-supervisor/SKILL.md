---
name: laravel-queues-and-supervisor
description: Guidance for working with Laravel/Lumen queue workers, AWS SQS FIFO queues, and Supervisor process management. Activate when working on background jobs, queue workers, scheduled tasks, Supervisor config files, or SQS integration.
metadata:
  tags: laravel, lumen, queues, sqs, supervisor, background-jobs
---

## Overview

Queue workers in these services use AWS SQS FIFO queues managed by Supervisor. The primary reference is `service-cron` (Lumen 5.5) and several Laravel services with similar patterns.

---

## SQS FIFO Queue Setup

Package: `shiftonelabs/laravel-sqs-fifo-queue`

Queue names follow a `sqs-{purpose}` convention:
- `sqs-cron` — scheduled/cron jobs
- `sqs-mail` — newsletter/email dispatch
- `sqs-priority` — high-priority tasks

**`.env` keys:**
```
QUEUE_DRIVER=sqs
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=eu-west-1
SQS_PREFIX=https://sqs.eu-west-1.amazonaws.com/
```

---

## Supervisor Configuration

Config files live in `resources/config/` and are deployed to `/etc/supervisor/conf.d/`.

**Pattern:**
```ini
[program:cron-workers]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/service/artisan queue:work sqs-cron --sleep=3 --tries=1
autostart=true
autorestart=true
numprocs=3
redirect_stderr=true
stdout_logfile=/var/log/supervisor/%(program_name)s.log
```

**Worker counts by purpose:**
- Cron workers: 3 processes
- Newsletter workers: 20 processes (high volume)
- AWS SES workers: 4 processes

Always set `--tries=1` for jobs that must not retry silently on failure — handle retries explicitly in the job.

---

## Job / Observer Pattern

Jobs in `service-cron` follow a `MessageHandler` + `Observer` split:

```php
// Handler receives message from queue
class MessageHandler
{
    public function handle(array $payload): void
    {
        $observer = $this->resolveObserver($payload['type']);
        $observer->process($payload);
    }
}

// Observers handle specific message types
namespace App\Http\Models\AwsSES\Queue\Observers;

class SendObserver
{
    public function process(array $payload): void
    {
        // handle SES send event
    }
}
```

Observer classes are named by action: `Send`, `Bounce`, `Reject`, `Open`, `Delivery`, `Click`, `Complaint`, `Snowflake`.

---

## Custom Artisan Queue Commands

For queues that need custom logic beyond `queue:work`, create dedicated Artisan commands:

```php
// Naming: Provider:queue:work
// Examples: AwsSES:queue:work, Contentful:queue:work
class AwsSESQueueWorkCommand extends Command
{
    protected $signature = 'AwsSES:queue:work';
    protected $description = 'Process AWS SES notification queue';

    public function handle(): void
    {
        $this->comment('Starting AwsSES queue worker...');
        // polling loop
    }
}
```

Register in `app/Console/Kernel.php`.

---

## Laravel Scheduler (non-Lumen services)

In full Laravel services, use the scheduler instead of raw cron:

```php
// app/Console/Kernel.php
protected function schedule(Schedule $schedule): void
{
    $schedule->job(new ExportJob)->hourly();
    $schedule->command('exports:run')->dailyAt('02:00');
}
```

Add to crontab on deployment:
```
* * * * * php /var/www/service/artisan schedule:run >> /dev/null 2>&1
```

---

## Redis Caching with Predis

For queue state tracking, use Redis via `predis/predis`:

```php
use Illuminate\Support\Facades\Cache;

Cache::store('redis')->put("job:{$id}:status", 'processing', now()->addMinutes(10));
$status = Cache::store('redis')->get("job:{$id}:status");
```

---

## Key Rules

- Always use `--tries=1` in Supervisor for jobs that must not silently retry
- Use FIFO queues when order matters (newsletters, sequential events)
- Log worker startup and errors to `/var/log/supervisor/`
- Supervisor programs use `%(process_num)02d` for numbered worker instances
- Place Supervisor configs in `resources/config/` (committed), deploy to `/etc/supervisor/conf.d/`
