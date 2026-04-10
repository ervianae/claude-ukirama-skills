---
name: sidekiq-patterns
description: Sidekiq background job patterns, Redis configuration, queue design, cron scheduling, retry strategies, and worker conventions for Rails applications.
origin: custom
---

# Sidekiq & Redis Patterns

Background job design, queue strategy, Redis usage, and worker conventions for Rails applications using Sidekiq 8+.

## When to Activate

- Creating new background workers
- Configuring Sidekiq queues or cron schedules
- Designing retry strategies
- Setting up Redis for caching, sessions, or rate limiting
- Debugging failed or stuck jobs
- Optimizing queue throughput

## Stack

| Tool | Version | Purpose |
|------|---------|---------|
| Sidekiq | 8.1+ | Background job processor |
| Sidekiq-Cron | latest | Cron-based job scheduling |
| Redis | 6+ | Job queue, cache, sessions, rate limiting |
| Redis Namespace | latest | Database isolation per concern |

---

## 1. Worker Structure

### Basic Worker

```ruby
# app/sidekiq_workers/email_notification_sidekiq_worker.rb
class EmailNotificationSidekiqWorker
  include Sidekiq::Job

  sidekiq_options queue: :email, retry: 1
  sidekiq_retry_in { |_count| 30 }

  def perform(params)
    followable_object = params['followable_object_type'].constantize.find(params['followable_object_id'])

    Ukirama::Notification::DocumentAction::GenerateEmailNotification.new(
      followable_object: followable_object,
      user_id: params['user_id'],
      notification_action: params['notification_action']
    ).run!
  end
end
```

### Rules

| Rule | Detail |
|------|--------|
| **Include `Sidekiq::Job`** | Not `Sidekiq::Worker` (deprecated in Sidekiq 8) |
| **Set explicit queue** | Always declare `sidekiq_options queue: :queue_name` |
| **Set retry strategy** | Explicitly set `retry: N`, `retry: true`, or `retry: false` |
| **Pass IDs, not objects** | Parameters are serialized to JSON — pass IDs and look up records |
| **Delegate to services** | Workers call DocumentAction services, never implement business logic |
| **One concern per worker** | Each worker does one thing |
| **Name convention** | `{Purpose}SidekiqWorker` — e.g., `EmailNotificationSidekiqWorker` |
| **File location** | `app/sidekiq_workers/` |

### Parameter Pattern

```ruby
# GOOD: Hash with string keys (JSON serialization safe)
EmailNotificationSidekiqWorker.perform_async(
  'followable_object_id' => document.id,
  'followable_object_type' => document.class.to_s,
  'user_id' => user.id,
  'notification_action' => 'completed'
)

# BAD: Symbol keys (may not survive JSON roundtrip in all cases)
EmailNotificationSidekiqWorker.perform_async(
  followable_object_id: document.id
)

# BAD: Passing ActiveRecord objects
EmailNotificationSidekiqWorker.perform_async(document)
```

---

## 2. Retry Strategies

### When to Retry

| Scenario | Retry | Rationale |
|----------|-------|-----------|
| Email/push notifications | `retry: 1` with 30s delay | Transient failures recoverable |
| Webhook delivery | `retry: true` (default 25) | External service may be temporarily down |
| Cloud storage upload | `retry: true` | Network issues are transient |
| Bulk import | `retry: false` | Partial imports cause data inconsistency |
| Report generation | `retry: false` | Idempotency not guaranteed |
| Database backup | `retry: false` | Custom retry logic inside worker |
| Scheduled maintenance | `retry: false` | Will run again on next schedule |

### Retry Configuration

```ruby
# Single retry with fixed delay
sidekiq_options retry: 1
sidekiq_retry_in { |_count| 30 }

# Default exponential backoff (25 retries over ~21 days)
sidekiq_options retry: true

# No retries
sidekiq_options retry: false

# Custom retry with backoff
sidekiq_options retry: 5
sidekiq_retry_in do |count|
  (count ** 4) + 15  # 16s, 31s, 96s, 271s, 640s
end
```

---

## 3. Queue Design

### Queue Configuration

```yaml
# config/sidekiq.yml
:verbose: false
:timeout: 30
:concurrency: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
:queues:
  - [critical, 3]
  - [default, 2]
  - [email, 1]
  - [notifications, 1]
  - [webhooks, 1]
  - [reports, 1]
  - [daily_backup, 1]
  - [upload_to_google_cloud_platform, 1]
```

### Queue Categories

| Category | Queues | Priority |
|----------|--------|----------|
| **Critical** | `critical` | High — payment, state changes |
| **Notifications** | `email`, `notifications` | Normal — user-facing alerts |
| **Integrations** | `webhooks` | Normal — external service calls |
| **Reports** | `reports` | Low — can wait |
| **Maintenance** | `daily_backup`, `delete_*`, `check_*`, `fix_*` | Low — scheduled tasks |
| **Recalculation** | `recalculate_*` | Low — dashboard/inventory recalcs |

### Rules

- Isolate queues by workload type to prevent starvation
- Use weighted priorities for critical vs background work
- Each worker declares its own queue explicitly
- Never use `:default` queue for important work

---

## 4. Cron Scheduling

### Schedule Configuration

```yaml
# config/schedule.yml
reports_sidekiq_worker:
  cron: "0 2 * * *"
  class: "ReportsSidekiqWorker"
  queue: reports

daily_backup_sidekiq_worker:
  cron: "0 2 * * *"
  class: "DailyBackupSidekiqWorker"
  queue: daily_backup

fix_solr_index_sidekiq_worker:
  cron: "*/5 * * * *"
  class: "FixSolrIndexSidekiqWorker"
  queue: fix_solr_index

monthly_delete_email_log:
  cron: "0 0 1 * *"
  class: "MonthlyDeleteEmailLogSidekiqWorker"
  queue: delete_api_request_log
```

### Schedule Timing Guide

| Frequency | Cron | Use Case |
|-----------|------|----------|
| Every 5 min | `*/5 * * * *` | Health checks, index fixes, session checks |
| Every 6 hours | `* */6 * * *` | Cache invalidation, email verification |
| Daily midnight | `0 0 * * *` | Cloud uploads, cleanup |
| Daily 1-3 AM | `0 1-3 * * *` | Backups, maintenance, storage checks |
| Daily 2 AM | `0 2 * * *` | Reports, recalculations, notifications |
| Daily 6 AM | `0 6 * * *` | Dashboard widgets (before business hours) |
| Monthly | `0 0 1 * *` | Log cleanup |

### Initializer (loads schedule on boot)

```ruby
# config/initializers/sidekiq.rb
Sidekiq.configure_server do |config|
  config.redis = { url: 'redis://localhost:6379/3' }

  config.on(:startup) do
    schedule = YAML.load_file(Rails.root.join('config', 'schedule.yml'))
    Sidekiq::Cron::Job.destroy_all!
    Sidekiq::Cron::Job.load_from_hash(schedule)
  end
end

Sidekiq.configure_client do |config|
  config.redis = { url: 'redis://localhost:6379/3' }
end
```

---

## 5. Redis Database Allocation

| Database | Purpose | TTL |
|----------|---------|-----|
| 3 | Sidekiq job queue | N/A (managed by Sidekiq) |
| 4 | Permission cache (dev/prod) + Rails cache | 1h (permissions), configurable (cache) |
| 5 | Permission cache (test) | 1h |

### Rules

- Never share Redis databases between concerns
- Document database allocation in a central location
- Use namespaces within databases for further isolation
- Always set TTL on cached data

---

## 6. Redis Caching Pattern

### Permission Cache

```ruby
module PermissionCache
  PERMISSIONS_TTL = 3600        # 1 hour
  ALL_PERMISSIONS_TTL = 86400   # 24 hours

  class << self
    def redis
      @redis ||= Redis.new(url: redis_url)
    end

    def fetch(user_id)
      cached = redis.get(cache_key(user_id))
      return JSON.parse(cached) if cached.present?

      permissions = load_from_database(user_id)
      redis.setex(cache_key(user_id), PERMISSIONS_TTL, permissions.to_json)
      permissions
    rescue Redis::BaseError => e
      Rails.logger.warn("[PermissionCache] Redis error: #{e.message}")
      load_from_database(user_id)  # Fallback to DB
    end

    def invalidate(user_id)
      redis.del(cache_key(user_id))
    rescue Redis::BaseError => e
      Rails.logger.warn("[PermissionCache] Error invalidating: #{e.message}")
    end
  end
end
```

### Rules

- Always handle `Redis::BaseError` — fall back to database
- Use `setex` (set with expiry), never `set` without TTL
- Invalidate on relevant model changes
- Log warnings on Redis failures, don't raise

---

## 7. Worker Invocation Patterns

### From Services (most common)

```ruby
# After import completes
ImportInAppNotificationSidekiqWorker.perform_async(
  'filename' => filename,
  'user_id' => user.id,
  'resource' => 'sales_invoices',
  'action' => 'import_data',
  'notification_action' => 'import_finished'
)
```

### From Other Workers (nested notifications)

```ruby
# Worker delegates notification to another worker
class FixedAssetSidekiqWorker
  include Sidekiq::Job
  sidekiq_options queue: :fixed_asset, retry: false

  def perform
    assets = FixedAssetDepreciationSchedule.pending
    assets.each do |asset|
      Ukirama::FixedAsset::DocumentAction::Post.new(asset: asset).run!
    end
    # Notify after batch completes
    AssetSidekiqNotifier.notify_completion(assets.count)
  end
end
```

### Scheduling for Later

```ruby
# Perform in 5 minutes
EmailNotificationSidekiqWorker.perform_in(5.minutes,
  'user_id' => user.id,
  'notification_action' => 'reminder'
)

# Perform at specific time
DailyBackupSidekiqWorker.perform_at(Date.tomorrow.beginning_of_day,
  'type' => 'full'
)
```

---

## 8. Complex Worker: Backup Pattern

```ruby
class DailyBackupSidekiqWorker
  include Sidekiq::Job
  sidekiq_options queue: :daily_backup, retry: false

  MAX_RETRIES = 3

  def perform
    dump_path = create_database_dump
    compressed_path = compress(dump_path)
    upload_with_retry(compressed_path)
  ensure
    cleanup_temp_files(dump_path, compressed_path)
  end

  private

  def upload_with_retry(path)
    retries = 0
    begin
      upload_to_remote(path)
    rescue UkiramaUtils::Errors::UploadToRemoteBackupFailed => e
      retries += 1
      retry if retries < MAX_RETRIES
      Rails.logger.error("[Backup] Failed after #{MAX_RETRIES} retries: #{e.message}")
    end
  end
end
```

### Rules for Complex Workers

- Use `ensure` to clean up temp files
- Implement custom retry logic for multi-step operations
- Log failures with context
- Never leave orphaned temp files

---

## 9. Code Review Checklist

### CRITICAL

- [ ] No ActiveRecord objects passed as parameters
- [ ] Worker delegates to service objects, not implementing business logic
- [ ] Redis database numbers don't conflict
- [ ] Retry strategy matches the operation type

### HIGH

- [ ] Queue explicitly declared in `sidekiq_options`
- [ ] Retry explicitly set (not relying on defaults)
- [ ] Parameters use string keys for JSON safety
- [ ] Cron schedule added to `schedule.yml` for scheduled workers
- [ ] Redis errors handled with fallback (not crashing)

### MEDIUM

- [ ] Worker naming follows `{Purpose}SidekiqWorker` convention
- [ ] File in `app/sidekiq_workers/`
- [ ] Temp files cleaned up in `ensure` block
- [ ] Logging present for error cases

## Related Skills

- `rails-erp-patterns` — Service object patterns (what workers delegate to)
- `ruby-testing` — Testing workers with RSpec
- `deployment-patterns` — Sidekiq process management in production
- `docker-patterns` — Containerizing Sidekiq alongside Rails
