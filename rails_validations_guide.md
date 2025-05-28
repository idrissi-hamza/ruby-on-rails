# Complete Ruby on Rails Validations Guide

A comprehensive guide to implementing robust validations in real-world Rails applications, using an e-commerce platform as an example.

## Table of Contents

1. [Overview](#overview)
2. [User Model Validations](#user-model-validations)
3. [Product Model Validations](#product-model-validations)
4. [Order Model Validations](#order-model-validations)
5. [Review Model Validations](#review-model-validations)
6. [Custom Validators](#custom-validators)
7. [Best Practices](#best-practices)
8. [Testing Validations](#testing-validations)

## Overview

Rails validations ensure data integrity by checking that model data meets specific criteria before being saved to the database. They run automatically before `save`, `create`, and `update` operations.

### Key Concepts

- **Model-level validations**: Defined in your ActiveRecord models
- **Database-level constraints**: Enforced at the database level for additional security
- **Conditional validations**: Rules that apply only under certain conditions
- **Custom validations**: Business-specific logic that can't be handled by built-in validators

## User Model Validations

The User model demonstrates authentication, authorization, and profile validation patterns.

### Basic Field Validations

```ruby
validates :first_name, :last_name, presence: true, length: { minimum: 2, maximum: 50 }
```

**Explanation**: Ensures names are present and within reasonable length limits. The `minimum: 2` prevents single-character entries, while `maximum: 50` prevents overly long entries that could cause display issues.

### Email Validation

```ruby
validates :email, presence: true, uniqueness: { case_sensitive: false }, 
          format: { with: URI::MailTo::EMAIL_REGEXP }
```

**Explanation**: 
- `presence: true` - Email is required
- `uniqueness: { case_sensitive: false }` - Prevents duplicate accounts with same email
- `format: { with: URI::MailTo::EMAIL_REGEXP }` - Uses Ruby's built-in email regex for format validation

### Password Security

```ruby
has_secure_password
validates :password, length: { minimum: 8 }, if: :password_digest_changed?
validates :password, format: { 
  with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/,
  message: "must include uppercase, lowercase, number and special character"
}, if: :password_digest_changed?
```

**Explanation**: 
- `has_secure_password` provides bcrypt encryption and basic password validation
- `if: :password_digest_changed?` only validates password when it's being set/changed
- Complex regex ensures strong passwords with mixed case, numbers, and special characters

### Age Validation with Custom Logic

```ruby
validates :date_of_birth, presence: true
validate :must_be_at_least_13_years_old

private

def must_be_at_least_13_years_old
  return unless date_of_birth
  
  if date_of_birth > 13.years.ago
    errors.add(:date_of_birth, "You must be at least 13 years old")
  end
end
```

**Explanation**: Custom validation method for COPPA compliance. Uses Rails' time helpers (`13.years.ago`) for readable date calculations.

### Conditional Business Logic

```ruby
validates :company_name, presence: true, if: :business_account?
validates :tax_id, presence: true, format: { with: /\A\d{2}-\d{7}\z/ }, 
          if: :business_account?

def business_account?
  account_type == 'business'
end
```

**Explanation**: Different validation rules based on account type. Business accounts require additional tax information.

### Data Normalization

```ruby
before_validation :normalize_email, :normalize_phone

def normalize_email
  self.email = email.downcase.strip if email.present?
end

def normalize_phone
  self.phone = phone.gsub(/\D/, '') if phone.present?
end
```

**Explanation**: Cleans data before validation runs. Email is lowercased and trimmed, phone numbers have non-digits removed.

## Product Model Validations

The Product model shows inventory management, pricing, and catalog validation patterns.

### SKU and Inventory Management

```ruby
validates :sku, presence: true, uniqueness: true, 
          format: { with: /\A[A-Z0-9\-]+\z/, message: "only allows uppercase letters, numbers, and hyphens" }
validates :stock_quantity, presence: true, numericality: { 
  greater_than_or_equal_to: 0,
  only_integer: true
}
```

**Explanation**: 
- SKU must be unique and follow a specific format for inventory systems
- Stock quantity must be a non-negative integer

### Price Validation with Business Logic

```ruby
validates :price, presence: true, numericality: { 
  greater_than: 0, 
  less_than: 1_000_000,
  precision: 2 
}

validate :price_must_be_reasonable_for_category

def price_must_be_reasonable_for_category
  return unless price && category
  
  case category.name.downcase
  when 'electronics'
    if price > 50_000
      errors.add(:price, "seems too high for electronics category")
    end
  when 'books'
    if price > 500
      errors.add(:price, "seems too high for books category")
    end
  end
end
```

**Explanation**: Combines standard numeric validation with category-specific business rules to catch data entry errors.

### Discount Logic Validation

```ruby
validate :discount_price_logic

def discount_price_logic
  return unless discount_price && price
  
  if discount_price >= price
    errors.add(:discount_price, "must be less than regular price")
  end
  
  discount_percentage = ((price - discount_price) / price) * 100
  if discount_percentage > 90
    errors.add(:discount_price, "discount cannot exceed 90%")
  end
end
```

**Explanation**: Ensures discount prices make business sense and prevents excessive discounts that might indicate errors.

### Availability and Stock Consistency

```ruby
validate :availability_matches_stock

def availability_matches_stock
  if stock_quantity == 0 && availability == 'in_stock'
    errors.add(:availability, "cannot be 'in stock' when quantity is 0")
  end
  
  if stock_quantity > 0 && availability == 'out_of_stock'
    errors.add(:availability, "cannot be 'out of stock' when quantity is available")
  end
end
```

**Explanation**: Prevents inconsistent inventory states that could confuse customers or cause ordering issues.

## Order Model Validations

The Order model demonstrates financial calculations, workflow states, and complex business rules.

### Financial Integrity

```ruby
validates :subtotal, :tax_amount, :shipping_cost, :total_amount,
          presence: true, numericality: { greater_than_or_equal_to: 0 }

validate :total_calculation_is_correct

def total_calculation_is_correct
  return unless subtotal && tax_amount && shipping_cost && total_amount
  
  calculated_total = subtotal + tax_amount + shipping_cost
  calculated_total -= discount_amount if discount_amount
  
  unless (calculated_total - total_amount).abs < 0.01
    errors.add(:total_amount, "does not match calculated total")
  end
end
```

**Explanation**: Ensures all financial calculations are mathematically correct. Uses floating-point comparison with tolerance (`0.01`) to handle precision issues.

### Workflow State Validation

```ruby
validates :status, inclusion: { 
  in: %w[pending confirmed processing shipped delivered cancelled refunded] 
}

validates :tracking_number, presence: true, if: :shipped_or_delivered?
validates :delivery_date, presence: true, if: :delivered?

def shipped_or_delivered?
  %w[shipped delivered].include?(status)
end
```

**Explanation**: Different fields are required based on order status. This ensures data completeness at each workflow stage.

### Business Rule Enforcement

```ruby
validate :user_can_place_order

def user_can_place_order
  if user.suspended?
    errors.add(:user, "account is suspended and cannot place orders")
  end
  
  if user.orders.where(payment_status: 'failed').count > 3
    errors.add(:user, "has too many failed payment attempts")
  end
end
```

**Explanation**: Prevents orders from users who shouldn't be able to purchase, helping with fraud prevention and account management.

### Payment Method Restrictions

```ruby
validate :payment_method_available_for_total

def payment_method_available_for_total
  if payment_method == 'cash_on_delivery' && total_amount > 1000
    errors.add(:payment_method, "Cash on delivery not available for orders over $1000")
  end
end
```

**Explanation**: Business rules about payment methods based on order value, common in e-commerce for risk management.

## Review Model Validations

The Review model shows user-generated content validation and purchase verification.

### Content Quality Validation

```ruby
validates :rating, presence: true, inclusion: { in: 1..5 }
validates :title, presence: true, length: { minimum: 5, maximum: 100 }
validates :content, presence: true, length: { minimum: 10, maximum: 2000 }
```

**Explanation**: Ensures reviews have meaningful content. Minimum lengths prevent spam, maximum lengths prevent overly long content.

### Duplicate Prevention

```ruby
validates :user_id, uniqueness: { scope: :product_id, 
                                 message: "can only review each product once" }
```

**Explanation**: Prevents users from submitting multiple reviews for the same product using scoped uniqueness.

### Purchase Verification

```ruby
validate :user_purchased_product

def user_purchased_product
  return if user.admin? # Admins can review without purchasing
  
  unless user.orders.joins(:order_items)
                   .where(order_items: { product_id: product_id })
                   .where(status: 'delivered').exists?
    errors.add(:base, "You can only review products you have purchased")
  end
end
```

**Explanation**: Ensures review authenticity by requiring actual purchase. Uses complex query to check across related models.

### Content Moderation

```ruby
validate :no_profanity_in_content

def no_profanity_in_content
  profanity_words = %w[badword1 badword2] # Would use a proper profanity filter
  
  if profanity_words.any? { |word| content.downcase.include?(word.downcase) }
    errors.add(:content, "contains inappropriate language")
  end
end
```

**Explanation**: Basic content filtering. In production, you'd use a more sophisticated profanity filter library.

## Custom Validators

Creating reusable validation logic for use across multiple models.

### Email Validator Class

```ruby
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?
    
    # Check format
    unless value.match?(URI::MailTo::EMAIL_REGEXP)
      record.errors.add(attribute, "is not a valid email format")
      return
    end
    
    # Check for disposable email domains
    disposable_domains = %w[tempmail.com 10minutemail.com]
    domain = value.split('@').last.downcase
    
    if disposable_domains.include?(domain)
      record.errors.add(attribute, "disposable email addresses are not allowed")
    end
    
    # Check for common typos in popular domains
    typo_corrections = {
      'gmail.co' => 'gmail.com',
      'gmail.cm' => 'gmail.com',
      'yahooo.com' => 'yahoo.com',
      'hotmial.com' => 'hotmail.com'
    }
    
    if typo_corrections.key?(domain)
      record.errors.add(attribute, "did you mean #{value.split('@').first}@#{typo_corrections[domain]}?")
    end
  end
end

# Usage in models:
validates :email, email: true
```

**Explanation**: Custom validator class that can be reused across models. Includes disposable email detection and typo correction suggestions for better user experience.

## Best Practices

### 1. Validation Order

Rails runs validations in this order:
1. `before_validation` callbacks
2. Built-in validations (presence, format, etc.)
3. Custom validation methods
4. `after_validation` callbacks

### 2. Error Message Quality

```ruby
validates :password, format: { 
  with: /complex_regex/,
  message: "must include uppercase, lowercase, number and special character"
}
```

Provide clear, actionable error messages that tell users exactly what's wrong and how to fix it.

### 3. Performance Considerations

```ruby
# Good: Scoped uniqueness validation
validates :name, uniqueness: { scope: :vendor_id }

# Good: Conditional expensive validations
validate :complex_business_rule, if: :needs_complex_validation?
```

Use scoped validations and conditional checks to avoid unnecessary database queries.

### 4. Data Normalization

```ruby
before_validation :normalize_data

def normalize_data
  self.email = email.downcase.strip if email.present?
  self.phone = phone.gsub(/\D/, '') if phone.present?
end
```

Clean and normalize data before validation to improve consistency.

### 5. Validation Contexts

```ruby
validates :password, presence: true, on: :create
validates :current_password, presence: true, on: :update
```

Use contexts to apply different validation rules for different operations.

## Testing Validations

### RSpec Examples

```ruby
RSpec.describe User, type: :model do
  describe 'validations' do
    subject { build(:user) }
    
    it { should validate_presence_of(:email) }
    it { should validate_uniqueness_of(:email).case_insensitive }
    it { should validate_length_of(:password).is_at_least(8) }
    
    context 'age validation' do
      it 'rejects users under 13' do
        user = build(:user, date_of_birth: 10.years.ago)
        expect(user).not_to be_valid
        expect(user.errors[:date_of_birth]).to include('You must be at least 13 years old')
      end
      
      it 'accepts users 13 and older' do
        user = build(:user, date_of_birth: 15.years.ago)
        expect(user).to be_valid
      end
    end
    
    context 'business account validations' do
      it 'requires company name for business accounts' do
        user = build(:user, account_type: 'business', company_name: nil)
        expect(user).not_to be_valid
        expect(user.errors[:company_name]).to include("can't be blank")
      end
      
      it 'does not require company name for personal accounts' do
        user = build(:user, account_type: 'personal', company_name: nil)
        expect(user).to be_valid
      end
    end
  end
end
```

### Testing Custom Validations

```ruby
describe 'custom validations' do
  it 'prevents orders from suspended users' do
    suspended_user = create(:user, status: 'suspended')
    order = build(:order, user: suspended_user)
    
    expect(order).not_to be_valid
    expect(order.errors[:user]).to include('account is suspended and cannot place orders')
  end
  
  it 'validates financial calculations' do
    order = build(:order, subtotal: 100, tax_amount: 8, shipping_cost: 10, total_amount: 999)
    
    expect(order).not_to be_valid
    expect(order.errors[:total_amount]).to include('does not match calculated total')
  end
end
```

## Common Patterns

### 1. State Machine Validations

```ruby
validates :status, inclusion: { in: %w[draft published archived] }
validate :cannot_publish_without_content

def cannot_publish_without_content
  if status == 'published' && content.blank?
    errors.add(:status, "cannot be published without content")
  end
end
```

### 2. Cross-Model Validations

```ruby
validate :shipping_address_deliverable

def shipping_address_deliverable
  return unless shipping_address
  
  unless shipping_address.deliverable?
    errors.add(:shipping_address, "is not in a deliverable area")
  end
end
```

### 3. Conditional Required Fields

```ruby
validates :phone, presence: true, if: :phone_required?

def phone_required?
  premium_member? || two_factor_enabled?
end
```

This comprehensive guide shows how to implement robust, production-ready validations in Rails applications. The key is to combine Rails' built-in validators with custom business logic to ensure data integrity, security, and a great user experience.