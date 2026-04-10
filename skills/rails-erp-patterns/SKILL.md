---
name: rails-erp-patterns
description: Ruby on Rails ERP patterns, conventions, and code review for the Ukirama ERP codebase. Covers DocumentAction services, validators, query objects, models, controllers, and API design specific to this project.
origin: custom
---

# Rails ERP Patterns & Code Review

Idiomatic Ruby/Rails patterns and project-specific conventions for the Ukirama ERP system. Use this skill when writing, reviewing, or refactoring Ruby code in this codebase.

## When to Activate

- Writing or modifying Ruby service objects, models, controllers, validators
- Reviewing Ruby/Rails code for quality and correctness
- Refactoring service objects or extracting concerns
- Designing new document actions or workflows
- Writing API endpoints or query objects
- Debugging state transition or advisory lock issues

## Architecture Overview

```
app/
├── apis/              # API service objects (BaseApi CRUD)
├── controllers/       # Web + API controllers
│   └── api/v1/        # Versioned API controllers
├── decorators/        # Draper decorators
├── form_permissions/  # Form-level permission checks
├── forms/             # Form objects (Ukirama::Form::Base)
├── grids/             # Solr-based grid definitions (UkiramaGrid::Definition)
├── histories/         # Paper Trail audit trail classes
├── models/            # ActiveRecord models + concerns
├── notifications/     # In-app notification system
├── params_definition/ # Parameter whitelisting/serialization
├── pdf_templates/     # PDF report templates (Prawn)
├── permissions/       # CSV-based permission definitions
├── reports/           # Report generation classes
├── select/            # Dropdown/select data providers
├── services/ukirama/  # Service objects (DocumentAction pattern)
├── sidekiq_workers/   # Background job classes
├── validators/        # Custom validator classes
├── webhooks/          # Webhook handlers
└── workflow_rules/    # Workflow authorization rules
lib/
├── ukirama_*/         # Query objects, utilities, calculators
```

---

## 1. Service Object Pattern (DocumentAction)

The core pattern in this codebase. All business logic flows through service objects.

### Base Class

```ruby
# Inherits from Ukirama::DocumentAction::Base
# Includes ActiveModel::Model
# Key methods: run!, run, process_run, raise_invalid_action!
```

### Correct Implementation

```ruby
class Ukirama::PurchaseOrder::DocumentAction::Complete < Ukirama::DocumentAction::Base
  attr_accessor :purchase_order, :skip_advisory_lock
  attr_reader :journal_entry

  def process_run
    with_advisory_lock_and_transaction(purchase_order, skip_advisory_lock: skip_advisory_lock) do
      if purchase_order_valid?
        purchase_order.complete!
        generate_movements!
        generate_accounting!
      else
        raise_invalid_action!(full_messages_from(purchase_order), purchase_order)
      end
      purchase_order
    end
  end

  private

  def purchase_order_valid?
    # Validation chain with short-circuit &&
    association_valid? &&
      fiscal_period_open? &&
      activate_valid? &&
      accounting_valid?
  end
end
```

### Rules

| Rule | Detail |
|------|--------|
| **Inherit correctly** | Always inherit from `Ukirama::DocumentAction::Base` |
| **Use `process_run`** | Never override `run!` — implement `process_run` |
| **Advisory lock** | Wrap mutations in `with_advisory_lock_and_transaction` |
| **Validation chain** | Use `&&` short-circuit in `*_valid?` methods |
| **Error raising** | Use `raise_invalid_action!(message, error_object)` |
| **Return the document** | `process_run` should return the primary document |
| **Use `save!` not `save`** | Always use bang save inside transactions — silent failures cause data inconsistency |
| **Inject dependencies** | Use `attr_accessor` for inputs, `attr_reader` for outputs |

### Anti-Patterns to Flag

```ruby
# BAD: save without bang — silently swallows failures
@order_document.save
# GOOD:
@order_document.save!

# BAD: overriding run! directly
def run!
  # custom logic
end
# GOOD: implement process_run
def process_run
  # custom logic
end

# BAD: initializing state in validation method
def document_valid?
  @movements = []  # Side effect in validation!
  # ...
end
# GOOD: initialize state in process_run, validate separately
def process_run
  @movements = []
  if document_valid?
    # ...
  end
end

# BAD: re-querying already loaded associations
sales_invoice.sales_invoice_lines.each  # N+1 if @sales_invoice_lines exists
# GOOD: use the pre-loaded instance variable
@sales_invoice_lines.each
```

### File Size Guidelines

| Threshold | Action |
|-----------|--------|
| < 400 lines | Ideal |
| 400-800 lines | Acceptable for complex documents |
| 800+ lines | Must extract — split into handler modules |

**Extraction targets for large services:**
- Stock movement logic -> `StockMovementHandler`
- Down payment logic -> `DownPaymentHandler`
- Billing calculations -> `BillingHandler`
- Lifecycle buy price movements -> dedicated service class
- Bonded zone logic -> `BondedZoneHandler`
- Activation validations -> config-driven validation arrays

---

## 2. Validator Pattern

### Custom Validator (DocumentAction)

```ruby
class UkiramaDocumentActionValidator::StockValidator
  def initialize
    @requests = []
    @errors = []
  end

  def add_validate_request!(item_id:, quantity:)
    @requests << ValidateRequest.new(item_id: item_id, quantity: quantity)
  end

  def valid?
    @requests.each { |req| validate(req) }
    @errors.empty?
  end

  def error_message
    # HTML-formatted error list
    "<ul>#{@errors.map { |e| "<li>#{e}</li>" }.join}</ul>"
  end

  private

  ValidateRequest = Struct.new(:item_id, :quantity, keyword_init: true)

  def validate(request)
    # validation logic, push to @errors on failure
  end
end
```

### ActiveModel Validator

```ruby
class ItemStockableValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    # standard Rails validator pattern
  end
end
```

### Rules

- Custom validators use `add_*!` then `valid?` then `error_message`
- Error messages are HTML-formatted with `<ul>/<li>` for bulk operations
- Use Struct for internal request/error objects
- I18n scopes follow: `[:ukirama_document_action_validator, :model_name]`
- Validator names end with `Validator` suffix

---

## 3. Model Conventions

```ruby
# == Schema Information (auto-generated by annotate gem)
# Table name: sales_invoices
# ...

class SalesInvoice < ApplicationRecord
  # 1. Constants
  DIRECT = 'direct'
  CONSIGNMENT = 'consignment'
  FULLY_BILLED = 'fully_billed'
  PARTIALLY_BILLED = 'partially_billed'

  # 2. Readonly attributes
  attr_readonly :code

  # 3. Concerns
  include UkiramaRevertableUnrejectableDefaultStateTransitions

  # 4. Associations
  belongs_to :partner
  belongs_to :warehouse
  has_many :sales_invoice_lines, dependent: :destroy

  # 5. Validations
  validates :code, presence: true, uniqueness: true

  # 6. Scopes
  scope :completed, -> { where(state: 'completed') }

  # 7. Callbacks (use sparingly)

  # 8. Paper Trail
  has_paper_trail

  # 9. Instance methods
end
```

### Rules

| Rule | Detail |
|------|--------|
| **Schema comments** | Keep `annotate` gem output at the top of model files |
| **Constants** | Define state/enum values as constants on the model |
| **attr_readonly** | Protect immutable fields like `code` |
| **State machine** | Use Transitions gem concerns, not raw string manipulation |
| **Auditing** | Include `has_paper_trail` on all important models |
| **Ordering** | Constants -> Concerns -> Associations -> Validations -> Scopes -> Callbacks -> Methods |
| **Boolean flags** | Use `is_` prefix: `is_decrease_stock`, `is_bonded_zone_transaction` |

---

## 4. Controller Patterns

### Web Controller

```ruby
class SalesInvoicesController < ApplicationController
  before_action :set_sales_invoice, only: [:show, :edit, :update]

  def index
    authorize_for!(action: :read, resource: SalesInvoice)
    # Grid-based response
  end

  def complete
    authorize_for!(action: :complete, resource: SalesInvoice)
    auth_permission_for!(document: @sales_invoice, workflow_rule: :complete)
    service = Ukirama::SalesInvoice::DocumentAction::Complete.new(
      sales_invoice: @sales_invoice
    )
    service.run!
    # respond
  rescue Ukirama::DocumentAction::ActionInvalid => e
    # error response
  end
end
```

### API Controller

```ruby
class Api::V1::SalesInvoicesController < Api::V1::BaseController
  # BaseController provides:
  # - skip_forgery_protection
  # - token authentication (param or header)
  # - rate limiting via Redis
  # - render_api_success / render_api_error helpers

  def create
    service = Ukirama::SalesInvoice::DocumentAction::Save.new(
      sales_invoice: build_sales_invoice
    )
    service.run!
    render_api_success(data: serialize(service.action_result))
  rescue Ukirama::DocumentAction::ActionInvalid => e
    render_api_error(message: e.message)
  end
end
```

### Rules

- Always call `authorize_for!` for permission checks
- Always call `auth_permission_for!` for workflow authorization
- Rescue `ActionInvalid` for user-facing error messages
- API responses use `render_api_success` / `render_api_error` envelope
- Never put business logic in controllers — delegate to service objects

---

## 5. Query Object Pattern

```ruby
# Located in lib/ukirama_*/
module UkiramaInventory
  class StockQuery
    def self.available_stock(warehouse_id:, item_id:, transaction_at:)
      sql = <<-SQL
        SELECT COALESCE(SUM(quantity), 0) as total
        FROM stock_movements
        WHERE warehouse_id = :warehouse_id
          AND item_id = :item_id
          AND transaction_at <= :transaction_at
      SQL
      result = ActiveRecord::Base.connection.select_one(
        ActiveRecord::Base.sanitize_sql_array([sql, {
          warehouse_id: warehouse_id,
          item_id: item_id,
          transaction_at: transaction_at
        }])
      )
      result['total'].to_d
    end
  end
end
```

### Rules

- Query objects live in `lib/` under a namespaced module
- Use class methods (`self.method_name`)
- Always use parameterized queries — **never** string interpolation in SQL
- Return Struct or simple values, not ActiveRecord objects
- Name pattern: `Ukirama{Domain}::Query` or `Ukirama{Domain}::{Specific}Query`

---

## 6. Concurrency & Advisory Locks

```ruby
# CORRECT: wrap in advisory lock
with_advisory_lock_and_transaction(document, skip_advisory_lock: skip_advisory_lock) do
  # mutations here
end

# The base class handles:
# - New records: skip lock, just wrap in transaction
# - Existing records: acquire advisory lock + transaction
# - Parent documents: acquire nested lock
# - Lock contention: raise ActionInvalid with 'other_ongoing_process_error'
```

### Rules

- All document state changes MUST use `with_advisory_lock_and_transaction`
- Pass `skip_advisory_lock: true` only when called from a parent that already holds the lock
- Lock key is `"#{class_name}#{code}"` — unique per document
- Never hold locks across HTTP requests

---

## 7. I18n Conventions

```ruby
# Service error scope
I18n.t('error_key', scope: ['activerecord', 'errors', 'models', 'sales_invoice'])

# Validator scope
I18n.t('error_key', scope: [:ukirama_document_action_validator, :validator_name])

# Model name
SalesInvoice.model_name.human  # Uses I18n automatically

# Supported locales: en, id (Indonesian)
```

### Rules

- All user-facing strings must use I18n
- Scope follows class hierarchy
- Never hardcode Indonesian or English strings in Ruby code
- Error messages may contain HTML (`<ul>`, `<li>`) for list formatting

---

## 8. Money & Currency

```ruby
# Currency comparison
if sales_invoice.currency == Money.default_currency.iso_code
  amount = value[:tax_amount]
  system_currency_amount = 0
else
  amount = value[:tax_amount]
  system_currency_amount = value[:tax_amount] * sales_invoice.exchange_rate
end
```

### Rules

- System currency is the company's base currency
- Foreign currency amounts always store both: `amount` + `system_currency_amount`
- Exchange rate multiplication for foreign -> system conversion
- Use `UkiramaMoney::Formatter.format(amount, currency)` for display
- Use `UkiramaUtils::DecimalRounder.round(value)` for precision

---

## 9. Background Jobs

```ruby
class ExportWorker
  include Sidekiq::Worker
  sidekiq_options queue: 'default', retry: 3

  def perform(document_id)
    document = SalesInvoice.find(document_id)
    # process
  end
end
```

### Rules

- Workers live in `app/sidekiq_workers/`
- Always pass IDs, never serialized objects
- Set explicit queue and retry options
- Use `sidekiq-cron` for scheduled jobs

---

## 10. Code Review Checklist (Rails-Specific)

### CRITICAL (block merge)

- [ ] No raw SQL string interpolation (use parameterized queries)
- [ ] No `save` without `!` inside transactions (use `save!`)
- [ ] Advisory lock used for all document state changes
- [ ] No hardcoded secrets or credentials
- [ ] No N+1 queries in loops (use `.includes()` or pre-loaded variables)

### HIGH (should fix)

- [ ] Service file under 800 lines
- [ ] No method over 50 lines
- [ ] No nesting deeper than 4 levels
- [ ] `Time.current` not `Time.now` (timezone correctness)
- [ ] Pre-loaded associations used instead of re-querying
- [ ] Error handling uses `raise_invalid_action!` not raw `raise`
- [ ] No commented-out code blocks

### MEDIUM (recommended)

- [ ] Predicate methods end with `?`
- [ ] No HTML in service objects (move to view/decorator layer)
- [ ] Repetitive validation blocks driven by config arrays
- [ ] State transitions use concern methods, not raw attribute assignment
- [ ] Constants defined on the model, not magic strings in services
- [ ] Instance variables initialized in `process_run`, not in `*_valid?`

### LOW (style)

- [ ] Consistent use of `if/else` vs ternary
- [ ] Frozen string literal comment present
- [ ] Model file follows ordering convention (Constants -> Concerns -> Associations -> ...)
- [ ] Schema annotations up to date

---

## 11. Common Bugs to Watch For

| Bug | Example | Fix |
|-----|---------|-----|
| **Division by zero** | `amount / total` where `total` can be 0 | Guard: `total.zero? ? 0 : amount / total` |
| **Silent save failure** | `record.save` inside transaction | Use `record.save!` |
| **Timezone mismatch** | `Time.now` vs `Time.current` | Always use `Time.current` or `Time.zone.now` |
| **N+1 in loops** | `record.association.each` inside a loop | Pre-load with `.includes()` or cache in instance variable |
| **Stale pre-loaded data** | Using `@lines` after records have been modified | Reload or re-query after mutations |
| **Lock ordering** | Acquiring locks in inconsistent order | Always lock parent before child |
| **Missing fiscal period check** | Completing without `fiscal_period_should_open?` | Include in validation chain |

---

## 12. Testing Conventions

```ruby
# spec/requests/api/v1/sales_invoices_spec.rb
RSpec.describe 'Api::V1::SalesInvoices', type: :request do
  let(:user) { create(:user) }
  let(:sales_invoice) { create(:sales_invoice) }

  before { sign_in user }

  describe 'POST /api/v1/sales_invoices' do
    it 'creates a sales invoice' do
      post api_v1_sales_invoices_path, params: valid_params
      expect(response).to have_http_status(:ok)
      expect(json_response['status']).to eq('success')
    end
  end
end
```

### Rules

- Use FactoryBot for test data (`create`, `build`, `build_stubbed`)
- Use `rswag` for API endpoint documentation + testing
- Transactional fixtures enabled — each test runs in a transaction
- Test service objects directly: `service.run!` and check `action_result`
- Test validators independently: `validator.valid?` and check `error_message`

---

## Related Skills

- `ecc:postgres-patterns` — Database query optimization, indexes
- `ecc:database-migrations` — Migration best practices
- `ecc:backend-patterns` — General API/service architecture
- `ecc:security-review` — Auth, injection, OWASP
- `ecc:api-design` — REST endpoint design
- `ecc:tdd-workflow` — Test-driven development
