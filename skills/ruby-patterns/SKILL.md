---
name: ruby-patterns
description: Idiomatic Ruby 3.4+ patterns, metaprogramming, error handling, concerns, form objects, and conventions for building maintainable Rails applications.
origin: custom
---

# Ruby Patterns & Idioms

Idiomatic Ruby patterns and best practices for Ruby 3.4+ and Rails 8+ applications. Covers language idioms, design patterns, error handling, and project conventions.

## When to Activate

- Writing new Ruby code
- Reviewing Ruby code for idioms and quality
- Refactoring Ruby classes or modules
- Designing new concerns, form objects, or utility modules
- Debugging Ruby-specific issues

## Ruby Version: 3.4+

Key changes in Ruby 3.4:
- `csv` and `ostruct` are no longer bundled — require explicit gem dependencies
- Pattern matching is stable
- Improved YJIT performance

---

## 1. Core Idioms

### Memoization

```ruby
# GOOD: Lazy initialization
def inventory_method
  @inventory_method ||= UkiramaQuery::SettingQuery.setting_with(key: :inventory_method)
end

# GOOD: Memoize nil-safe with defined?
def cached_result
  return @cached_result if defined?(@cached_result)
  @cached_result = expensive_query
end

# BAD: Re-computing on every call
def inventory_method
  UkiramaQuery::SettingQuery.setting_with(key: :inventory_method)
end
```

### Safe Navigation & Presence

```ruby
# GOOD
sales_invoice.source_document&.code
partner.credit_limit.presence || 0

# GOOD: try for method delegation
sales_invoice.try(:source_document)

# BAD: Explicit nil checks when avoidable
if sales_invoice.source_document != nil
  sales_invoice.source_document.code
end
```

### Collection Operations

```ruby
# GOOD: Functional collection methods
item_ids = lines.map(&:item_id).uniq.compact
active_items = items.select(&:active?)
totals = lines.sum(&:amount)
items_by_id = items.index_by(&:id)
grouped = movements.group_by(&:stock_movement)

# GOOD: each.with_index for row tracking
lines.each.with_index(1) do |line, index|
  # index starts at 1 for user-facing row numbers
end

# BAD: Manual iteration with counter
index = 0
lines.each do |line|
  index += 1
end
```

### String Formatting

```ruby
# GOOD: Interpolation
"#{partner.code} - #{partner.name}"

# GOOD: Frozen string for constants
THUMBNAIL_SIZE = '200x200'.freeze
DATE_FORMAT = '%Y-%m-%d'.freeze

# GOOD: Heredoc for SQL/long strings
sql = <<-SQL
  SELECT COALESCE(SUM(quantity), 0) as total
  FROM stock_movements
  WHERE warehouse_id = :warehouse_id
SQL

# BAD: String concatenation
partner.code + " - " + partner.name
```

### Conditional Assignment

```ruby
# GOOD: Ternary for simple cases
state = total > limit ? :over_billed : :partially_billed

# GOOD: if/else for complex cases
billing_state = if invoice_total > order_total
                  SalesInvoice::OVER_BILLED
                elsif invoice_total == order_total
                  SalesInvoice::FULLY_BILLED
                else
                  SalesInvoice::PARTIALLY_BILLED
                end

# GOOD: Guard clause / early return
def open_billing_valid?
  return true if sales_invoice.is_order_document
  return false if @order_document.is_billing_closed
  true
end
```

---

## 2. ActiveSupport::Concern Pattern

### State Machine Concern

```ruby
# app/models/concerns/ukirama_active_inactive_state_transitions.rb
module UkiramaActiveInactiveStateTransitions
  extend ActiveSupport::Concern

  ACTIVE = :active
  INACTIVE = :inactive

  included do
    include ActiveModel::Transitions

    state_machine do
      state INACTIVE
      state ACTIVE

      event :activate do
        transitions to: ACTIVE, from: INACTIVE
      end

      event :deactivate do
        transitions to: INACTIVE, from: ACTIVE
      end
    end

    searchable do
      integer :id
      string :state
      time :created_at
      time :updated_at
    end
  end

  module ClassMethods
    def all_states
      [ACTIVE, INACTIVE]
    end
  end
end
```

### Rules

| Rule | Detail |
|------|--------|
| **Use `extend ActiveSupport::Concern`** | Not bare `Module` with `self.included` |
| **Use `included` block** | For macros (state_machine, searchable, validations) |
| **Use `ClassMethods` module** | For class-level methods — auto-extended by Concern |
| **Define constants in the concern** | State values, enums live in the concern |
| **One concern per file** | Named to match the module |
| **File location** | `app/models/concerns/` |

---

## 3. Form Object Pattern

### Base Class

```ruby
# app/forms/ukirama/form/base.rb
class Ukirama::Form::Base
  include ActiveModel::Model
  include Virtus.model
  include Draper::Decoratable

  DATETIME_FORMAT = '%Y-%m-%d %H:%M:%S %z'.freeze
  DATE_FORMAT = '%Y-%m-%d'.freeze

  def initialize(...)
    setup
    super
  end

  def safe_assign_attributes(attributes_hash)
    assign_attributes(attributes_hash)
  end
end
```

### Form Implementation

```ruby
class SalesInvoiceForm < Ukirama::Form::Base
  SCOPE = ['activemodel', 'errors', 'models', 'sales_invoice_form'].freeze

  # Virtus typed attributes
  attribute :document_id, Integer
  attribute :partner_id, Integer
  attribute :quantity, BigDecimal
  attribute :transaction_at, UkiramaUtils::Virtus::TimeWithZoneAttribute
  attribute :lines, Array[SalesInvoiceLineForm]

  def setup
    # Called before super in initialize
  end

  def valid_for_complete?
    validate_partner &&
      validate_lines &&
      validate_amounts
  end

  private

  def validate_partner
    return true if partner_id.present?
    errors.add(:partner_id, I18n.t('required', scope: SCOPE))
    false
  end
end
```

### Rules

- Inherit from `Ukirama::Form::Base`
- Use Virtus `attribute` for typed fields with coercion
- Nested forms via `Array[ChildFormClass]`
- Define `SCOPE` constant for i18n error messages
- Implement `setup` hook for initialization logic
- Use `each.with_index` for row-number error tracking

---

## 4. Error Handling

### Custom Exception Classes

```ruby
# Namespaced under feature modules
module Ukirama
  module Permission
    module Errors
      class AbilityNotDefined < RuntimeError; end
      class SubjectNotFound < RuntimeError; end
      class AccessDenied < RuntimeError; end
    end
  end
end

# With rich context
class Ukirama::DocumentAction::ActionInvalid < StandardError
  attr_reader :error_object

  def initialize(message)
    @message = message[:message]
    @error_object = message[:error_object]
  end

  def message
    @message
  end
end

# Simple domain errors
module UkiramaApi::Errors
  class ApiAccessTokenNotFound < RuntimeError; end
  class ExceedsConcurrentUserLimit < RuntimeError; end
  class ExceedsRequestRateLimit < RuntimeError; end
end
```

### Exception Handling Patterns

```ruby
# GOOD: Specific rescue with fallback
def fetch_permissions(user_id)
  cached = redis.get(key)
  JSON.parse(cached)
rescue Redis::BaseError, JSON::ParserError => e
  Rails.logger.warn("[PermissionCache] Error: #{e.message}")
  load_from_database(user_id)
end

# GOOD: Rescue with ensure cleanup
def process_backup
  dump_path = create_dump
  upload(dump_path)
rescue => e
  Rails.logger.error("[Backup] Failed: #{e.message}")
  raise
ensure
  FileUtils.rm_f(dump_path) if dump_path
end

# BAD: Bare rescue (catches everything including syntax errors)
begin
  dangerous_operation
rescue
  # swallows all errors
end

# BAD: Rescue without logging or re-raising
begin
  dangerous_operation
rescue StandardError
  nil
end
```

### Rules

- Inherit from `StandardError` for business errors, `RuntimeError` for system errors
- Namespace exceptions under their feature module
- Include context (error_object, message) in custom exceptions
- Always log warnings/errors before fallback
- Use `ensure` for cleanup, not `rescue`
- Rescue specific exceptions, not bare `rescue`

---

## 5. Query Object Pattern

### Structure

```ruby
# lib/ukirama_inventory/stock_query.rb
class UkiramaInventory::StockQuery
  # Use Struct for return values
  StockSummary = Struct.new(:item_id, :warehouse_id, :quantity, :available, keyword_init: true)

  def self.available_stock(warehouse_id:, item_id:, transaction_at:)
    sql = <<-SQL
      SELECT COALESCE(SUM(quantity), 0) as total
      FROM stock_movements
      WHERE warehouse_id = :warehouse_id
        AND item_id = :item_id
        AND transaction_at <= :transaction_at
    SQL

    sanitized = ActiveRecord::Base.sanitize_sql_array([sql, {
      warehouse_id: warehouse_id,
      item_id: item_id,
      transaction_at: transaction_at
    }])

    result = ActiveRecord::Base.connection.select_one(sanitized)
    result['total'].to_d
  end
end
```

### Rules

- Live in `lib/` under namespaced modules
- Use class methods (`self.method_name`)
- Return Structs or simple values, not ActiveRecord objects
- Always use `sanitize_sql_array` — never string interpolation in SQL
- Define Struct at class level for return types
- Use `keyword_init: true` for readable Struct creation

---

## 6. Module Utility Pattern

### Singleton Module

```ruby
module PermissionCache
  DEFAULT_REDIS_URLS = {
    'development' => 'redis://localhost:6379/4',
    'production' => 'redis://localhost:6379/4'
  }.freeze

  class << self
    def redis
      @redis ||= Redis.new(url: redis_url)
    end

    def fetch(user_id)
      # ...
    end

    def invalidate(user_id)
      # ...
    end

    private

    def redis_url
      DEFAULT_REDIS_URLS[Rails.env]
    end
  end
end
```

### Static Utility

```ruby
module UkiramaUtils
  module DecimalRounder
    def self.round(value, precision: 10)
      value.round(precision)
    end
  end
end

module UkiramaFormatter
  module DateTimeFormatter
    def self.format_datetime(datetime)
      datetime&.strftime('%Y-%m-%d %H:%M:%S')
    end
  end
end
```

### Rules

- Use `class << self` for modules with state (Redis connections, etc.)
- Use `def self.method` for pure utility methods
- Freeze constant hashes and arrays
- Use `@var ||=` for lazy-initialized connections

---

## 7. Money & Currency

```ruby
# Configuration
Money.default_currency = UkiramaUtils::Config.default_currency
Money.disallow_currency_conversion!
Money.locale_backend = :i18n
Money.rounding_mode = BigDecimal::ROUND_HALF_EVEN

# Usage pattern: dual-amount storage
if sales_invoice.currency == Money.default_currency.iso_code
  amount = value[:tax_amount]
  system_currency_amount = 0
else
  amount = value[:tax_amount]
  system_currency_amount = value[:tax_amount] * sales_invoice.exchange_rate
end

# Formatting
UkiramaMoney::Formatter.format(amount, currency)

# Rounding
UkiramaUtils::DecimalRounder.round(quantity)
```

---

## 8. I18n Patterns

```ruby
# Scoped translations
I18n.t('error_key', scope: ['activerecord', 'errors', 'models', 'sales_invoice'])

# Model human name (auto-i18n)
SalesInvoice.model_name.human

# Attribute human name
SalesInvoice.human_attribute_name(:transaction_at)

# Interpolation
I18n.t('balance_not_enough_detail',
  scope: ['activerecord', 'errors', 'models', 'sales_invoice'],
  sales_down_payment_balance: formatted_balance,
  amount: formatted_amount
)
```

### Rules

- All user-facing strings MUST use I18n
- Supported locales: `en`, `id` (Indonesian)
- Scope pattern follows class hierarchy
- Use symbol keys for translation lookup

---

## 9. Rubocop Configuration

```yaml
# .rubocop.yml
AllCops:
  TargetRubyVersion: 3.1
  DisabledByDefault: true  # Opt-in only
  Include:
    - 'app/services/ukirama/**/*'
    - 'lib/**/*.rb'
  Exclude:
    - 'lib/tasks/**/*'

Style/TrailingCommaInArrayLiteral:
  Enabled: true
  EnforcedStyleForMultiline: comma

Style/TrailingCommaInHashLiteral:
  Enabled: true
  EnforcedStyleForMultiline: comma

Layout/IndentationWidth:
  Width: 2
```

Key conventions:
- 2-space indentation
- Trailing commas in multiline arrays/hashes
- Opt-in linting (only explicitly enabled cops)
- Focus on `app/services/` and `lib/`

---

## 10. Code Review Checklist

### CRITICAL

- [ ] No SQL string interpolation — use `sanitize_sql_array`
- [ ] No bare `rescue` — always specify exception class
- [ ] No hardcoded secrets or credentials
- [ ] Custom exceptions have meaningful context

### HIGH

- [ ] Memoization used for expensive lookups
- [ ] `Time.current` not `Time.now`
- [ ] Collection methods used (map, select, index_by) not manual loops
- [ ] Frozen string constants
- [ ] Specific exception rescue with logging

### MEDIUM

- [ ] Guard clauses / early returns for readability
- [ ] Structs for value objects in query returns
- [ ] Concerns follow `ActiveSupport::Concern` pattern
- [ ] Form objects use Virtus typed attributes
- [ ] I18n for all user-facing strings

### LOW

- [ ] Trailing commas in multiline structures
- [ ] String interpolation over concatenation
- [ ] `&.` safe navigation over `try`
- [ ] Consistent formatting (2-space indent)

## Related Skills

- `rails-erp-patterns` — Rails service object and validator patterns
- `ruby-testing` — RSpec and FactoryBot patterns
- `sidekiq-patterns` — Background job patterns
- `postgres-patterns` — Database query optimization
