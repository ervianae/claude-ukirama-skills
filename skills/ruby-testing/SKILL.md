---
name: ruby-testing
description: Ruby testing patterns with RSpec, FactoryBot, RSwag/OpenAPI, shared examples, and request specs. TDD methodology for Rails service objects, validators, and API endpoints.
origin: custom
---

# Ruby Testing Patterns

RSpec + FactoryBot + RSwag testing patterns for Ruby on Rails applications. Covers TDD workflow, factory design, API documentation testing, and service object testing.

## When to Activate

- Writing new RSpec tests
- Setting up test infrastructure
- Creating or modifying factories
- Writing API request specs with RSwag/OpenAPI
- Debugging test failures
- Reviewing test coverage and quality

## Test Stack

| Tool | Purpose |
|------|---------|
| RSpec Rails | Test framework |
| FactoryBot Rails | Test data factories |
| RSwag Specs | OpenAPI 3.0.1 documentation + request testing |
| Database Cleaner | Database state management |
| Parallel Tests | Multi-process test execution |
| Minitest | Legacy/simple unit tests |

## Directory Structure

```
spec/
├── rails_helper.rb                    # RSpec Rails config
├── spec_helper.rb                     # Core RSpec config
├── swagger_helper.rb                  # RSwag/OpenAPI config
├── requests/
│   └── api/v1/
│       └── client_api/                # API request specs (RSwag)
├── services/                          # Service object specs
├── models/                            # Model specs
├── lib/                               # Library specs
├── support/
│   └── shared_examples/
│       └── api_responses.rb           # Shared API response examples
└── swagger/
    └── components/
        └── schemas/                   # OpenAPI schema definitions

test/
└── factories/                         # FactoryBot factory definitions
```

Note: Factories live in `test/factories/`, not `spec/factories/`.

---

## 1. RSpec Configuration

### rails_helper.rb

```ruby
require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require_relative '../config/environment'

abort("The Rails environment is running in production mode!") if Rails.env.production?
require 'rspec/rails'

# Auto-load all support files
Dir[Rails.root.join('spec', 'support', '**', '*.rb')].sort.each { |f| require f }

RSpec.configure do |config|
  config.fixture_paths = [Rails.root.join('spec/fixtures')]
  config.use_transactional_fixtures = true
  config.infer_spec_type_from_file_location!
  config.filter_rails_from_backtrace!
end
```

### spec_helper.rb

```ruby
RSpec.configure do |config|
  config.expect_with :rspec do |expectations|
    expectations.include_chain_clauses_in_custom_matcher_descriptions = true
  end

  config.mock_with :rspec do |mocks|
    mocks.verify_partial_doubles = true
  end

  config.shared_context_metadata_behavior = :apply_to_host_groups
end
```

---

## 2. Factory Patterns

### Basic Factory

```ruby
FactoryBot.define do
  factory :warehouse do
    sequence(:code) { |n| "WH-#{n}" }
    name { "Warehouse Name" }
    association :warehouse_category

    to_create do |warehouse|
      Ukirama::Warehouse::DocumentAction::Save.new(warehouse: warehouse).run!
      Ukirama::Warehouse::DocumentAction::Activate.new(warehouse: warehouse).run!
    end
  end
end
```

### Rules

| Rule | Detail |
|------|--------|
| **Use `to_create`** | Factories MUST use DocumentAction services for persistence, not raw `save!` |
| **Use `sequence`** | All unique fields (code, name) must use sequences |
| **Activate after save** | Resources that need active state must call both Save and Activate |
| **Use associations** | Prefer `association :parent` over manual ID assignment |
| **Keep minimal** | Only set required attributes — override in tests for specific scenarios |

### Factory with Traits

```ruby
FactoryBot.define do
  factory :sales_invoice do
    sequence(:code) { |n| "SI-#{n}" }
    association :partner
    association :warehouse
    association :company
    state { 'drafted' }

    trait :completed do
      to_create do |invoice|
        Ukirama::SalesInvoice::DocumentAction::Save.new(sales_invoice: invoice).run!
        Ukirama::SalesInvoice::DocumentAction::Complete.new(sales_invoice: invoice).run!
      end
    end

    trait :with_lines do
      after(:create) do |invoice|
        create_list(:sales_invoice_line, 3, sales_invoice: invoice)
      end
    end

    to_create do |invoice|
      Ukirama::SalesInvoice::DocumentAction::Save.new(sales_invoice: invoice).run!
    end
  end
end
```

### Anti-Patterns

```ruby
# BAD: Direct save bypasses business logic
to_create { |record| record.save! }

# GOOD: Use DocumentAction services
to_create do |record|
  Ukirama::Record::DocumentAction::Save.new(record: record).run!
end

# BAD: Hardcoded IDs
factory :item do
  warehouse_id { 1 }
end

# GOOD: Use associations
factory :item do
  association :warehouse
end
```

---

## 3. Request Specs with RSwag

### Basic API Spec

```ruby
require 'swagger_helper'

RSpec.describe 'Api::V1::ClientApi::Items', type: :request do
  let(:user) { create(:user) }

  before { sign_in user }

  path '/api/v1/client_api/items' do
    get 'List items' do
      tags 'Items'
      produces 'application/json'
      parameter name: :locale, in: :query, type: :string, example: 'en'
      parameter name: :page, in: :query, type: :integer, required: false

      response '200', 'items found' do
        schema type: :object,
          properties: {
            status: {
              type: :object,
              properties: {
                code: { type: :integer },
                message: { type: :string }
              }
            },
            data: {
              type: :array,
              items: { type: :object }
            }
          }

        let(:locale) { 'en' }

        after do |example|
          example.metadata[:response][:content] = {
            'application/json' => { example: JSON.parse(response.body, symbolize_names: true) }
          }
        end

        run_test!
      end

      response '401', 'unauthorized' do
        include_examples 'an unauthorized response'
      end
    end
  end
end
```

### RSwag Rules

- Always `require 'swagger_helper'` (not rails_helper)
- Use `tags` for Swagger grouping
- Define `parameter` for all query/path/body params
- Include `locale` parameter on all endpoints
- Use `after` block to capture response examples
- Use shared examples for common error responses (401, 403, 404, 500)
- Run `run_test!` to execute the request and validate

### Swagger Helper Configuration

```ruby
# spec/swagger_helper.rb
RSpec.configure do |config|
  config.openapi_root = Rails.root.join('swagger').to_s

  config.openapi_specs = {
    'v1/swagger.yaml' => {
      openapi: '3.0.1',
      info: { title: 'API V1', version: 'v1' },
      paths: {},
      servers: [
        { url: 'http://localhost:3000' }
      ]
    }
  }
  config.openapi_format = :yaml
end
```

---

## 4. Shared Examples

### API Response Shared Examples

```ruby
# spec/support/shared_examples/api_responses.rb

RSpec.shared_examples 'an unauthorized response' do
  response '401', 'unauthorized' do
    schema type: :object,
      properties: {
        status: {
          type: :object,
          properties: {
            code: { type: :integer },
            message: { type: :string }
          }
        },
        errors: { type: :array, items: { type: :string } }
      }
    run_test!
  end
end

RSpec.shared_examples 'a not found response' do |error_message|
  response '404', 'not found' do
    schema type: :object,
      properties: {
        status: {
          type: :object,
          properties: {
            code: { type: :integer },
            message: { type: :string, example: error_message }
          }
        }
      }
    run_test!
  end
end

RSpec.shared_examples 'a forbidden response' do |message|
  response '403', 'forbidden' do
    schema type: :object,
      properties: {
        status: {
          type: :object,
          properties: {
            code: { type: :integer },
            message: { type: :string, example: message || 'Forbidden' }
          }
        }
      }
    run_test!
  end
end
```

### Usage

```ruby
response '401', 'unauthorized' do
  include_examples 'an unauthorized response'
end

response '404', 'not found' do
  include_examples 'a not found response', 'Item not found'
end
```

---

## 5. Service Object Testing

### Testing DocumentAction Services

```ruby
RSpec.describe Ukirama::SalesInvoice::DocumentAction::Complete do
  let(:sales_invoice) { create(:sales_invoice, :with_lines) }

  describe '#run!' do
    context 'when valid' do
      it 'completes the sales invoice' do
        service = described_class.new(sales_invoice: sales_invoice)
        result = service.run!

        expect(result.state).to eq('completed')
      end

      it 'generates journal entry' do
        service = described_class.new(sales_invoice: sales_invoice)
        service.run!

        expect(service.journal_entry).to be_present
      end
    end

    context 'when invalid' do
      it 'raises ActionInvalid' do
        sales_invoice.partner = nil

        service = described_class.new(sales_invoice: sales_invoice)

        expect { service.run! }.to raise_error(
          Ukirama::DocumentAction::ActionInvalid
        )
      end
    end
  end

  describe '#run' do
    context 'when invalid' do
      it 'returns false without raising' do
        sales_invoice.partner = nil

        service = described_class.new(sales_invoice: sales_invoice)
        result = service.run

        expect(result).to be false
        expect(service.errors[:base]).to be_present
      end
    end
  end
end
```

### Testing Validators

```ruby
RSpec.describe UkiramaDocumentActionValidator::CustomerCreditLimitValidator do
  let(:partner) { create(:partner) }

  describe '#valid?' do
    it 'returns true when within credit limit' do
      validator = described_class.new(
        resource: sales_invoice,
        partner: partner,
        customer_credit_limit_validation_type: 'warn'
      )

      expect(validator).to be_valid
    end

    it 'returns false with error message when exceeds limit' do
      partner.update!(credit_limit: 0)

      validator = described_class.new(
        resource: sales_invoice,
        partner: partner,
        customer_credit_limit_validation_type: 'block'
      )

      expect(validator).not_to be_valid
      expect(validator.error_message).to include('credit limit')
    end
  end
end
```

---

## 6. OpenAPI Schema Definitions

```ruby
# spec/swagger/components/schemas/error_schema.rb
module ErrorSchema
  def self.generic_error
    {
      type: :object,
      properties: {
        status: {
          type: :object,
          properties: {
            code: { type: :integer },
            message: { type: :string }
          }
        },
        errors: {
          type: :array,
          items: { type: :string }
        }
      }
    }
  end

  def self.not_found(resource_name)
    {
      type: :object,
      properties: {
        status: {
          type: :object,
          properties: {
            code: { type: :integer, example: 404 },
            message: { type: :string, example: "#{resource_name} not found" }
          }
        }
      }
    }
  end
end
```

---

## 7. TDD Workflow

### Red-Green-Refactor

```
1. Write the test first (RED)
   - Define expected behavior
   - Run test — it should FAIL

2. Write minimal implementation (GREEN)
   - Only enough code to pass the test
   - Run test — it should PASS

3. Refactor (IMPROVE)
   - Clean up without changing behavior
   - Run test — it should still PASS
```

### Test Naming Convention

```ruby
# Describe the class/method
RSpec.describe Ukirama::SalesInvoice::DocumentAction::Complete do
  # Context describes the scenario
  context 'when sales invoice has decrease stock enabled' do
    # It describes the expected behavior
    it 'generates stock movements for each line' do
      # Arrange - Act - Assert
    end
  end
end
```

### What to Test

| Layer | What to Test | How |
|-------|-------------|-----|
| Service objects | `run!` success/failure, side effects | Direct instantiation + `run!` |
| Validators | `valid?` returns, `error_message` content | Direct instantiation + assertions |
| API endpoints | Response status, body schema, auth | RSwag request specs |
| Models | Validations, scopes, associations | Model specs |
| Query objects | Return values, SQL correctness | Direct class method calls |

---

## 8. Running Tests

```bash
# Run all specs
bundle exec rspec

# Run specific file
bundle exec rspec spec/requests/api/v1/client_api/items_spec.rb

# Run specific test by line
bundle exec rspec spec/requests/api/v1/client_api/items_spec.rb:42

# Run with tag
bundle exec rspec --tag focus

# Generate Swagger docs
bundle exec rake rswag:specs:swaggerize

# Run in parallel
bundle exec parallel_rspec spec/
```

---

## 9. Test Review Checklist

- [ ] Tests use factories with `to_create` DocumentAction pattern
- [ ] API specs use RSwag with proper schema definitions
- [ ] Shared examples used for common responses (401, 403, 404)
- [ ] Service tests cover both `run!` (raises) and `run` (returns false) paths
- [ ] Locale parameter included in API specs
- [ ] No direct database manipulation — use factories and services
- [ ] Tests are isolated — no dependency on execution order
- [ ] Error cases tested with specific error message assertions

## Related Skills

- `rails-erp-patterns` — Service object and validator patterns
- `tdd-workflow` — General TDD methodology
- `e2e-testing` — Playwright E2E patterns
- `api-design` — REST API design standards
