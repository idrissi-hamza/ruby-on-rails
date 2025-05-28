# Complete Rails Validations - Real Project Examples

This document contains comprehensive Rails validation examples from a real e-commerce application, demonstrating production-ready validation patterns.

## User Model

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :orders, dependent: :destroy
  has_many :addresses, dependent: :destroy
  has_many :reviews, dependent: :destroy
  has_one :cart, dependent: :destroy
  
  # Basic validations
  validates :first_name, :last_name, presence: true, length: { minimum: 2, maximum: 50 }
  validates :email, presence: true, uniqueness: { case_sensitive: false }, 
            format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :phone, format: { with: /\A[\+]?[1-9][\d]{0,15}\z/, message: "Invalid phone format" },
            allow_blank: true
  
  # Password validations (assuming you're using has_secure_password)
  has_secure_password
  validates :password, length: { minimum: 8 }, if: :password_digest_changed?
  validates :password, format: { 
    with: /\A(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/,
    message: "must include uppercase, lowercase, number and special character"
  }, if: :password_digest_changed?
  
  # Status and role validations
  validates :status, inclusion: { in: %w[active inactive suspended] }
  validates :role, inclusion: { in: %w[customer admin moderator] }
  
  # Age validation with business logic
  validates :date_of_birth, presence: true
  validate :must_be_at_least_13_years_old
  validate :reasonable_birth_date
  
  # Email confirmation for registration
  validates :email_confirmed, acceptance: true, on: :create, unless: :admin?
  
  # Custom validations
  validate :unique_email_across_deleted_users
  validate :phone_required_for_premium_users
  
  # Conditional validations
  validates :company_name, presence: true, if: :business_account?
  validates :tax_id, presence: true, format: { with: /\A\d{2}-\d{7}\z/ }, 
            if: :business_account?
  
  # Callbacks for additional validation logic
  before_validation :normalize_email, :normalize_phone
  
  private
  
  def must_be_at_least_13_years_old
    return unless date_of_birth
    
    if date_of_birth > 13.years.ago
      errors.add(:date_of_birth, "You must be at least 13 years old")
    end
  end
  
  def reasonable_birth_date
    return unless date_of_birth
    
    if date_of_birth < 120.years.ago
      errors.add(:date_of_birth, "Birth date seems unrealistic")
    end
    
    if date_of_birth > Date.current
      errors.add(:date_of_birth, "Birth date cannot be in the future")
    end
  end
  
  def unique_email_across_deleted_users
    deleted_user = User.only_deleted.find_by(email: email)
    if deleted_user && deleted_user != self
      errors.add(:email, "This email was previously used by a deleted account")
    end
  end
  
  def phone_required_for_premium_users
    if premium_member? && phone.blank?
      errors.add(:phone, "is required for premium members")
    end
  end
  
  def business_account?
    account_type == 'business'
  end
  
  def premium_member?
    membership_tier == 'premium'
  end
  
  def admin?
    role == 'admin'
  end
  
  def normalize_email
    self.email = email.downcase.strip if email.present?
  end
  
  def normalize_phone
    self.phone = phone.gsub(/\D/, '') if phone.present?
  end
end
```

## Product Model

```ruby
# app/models/product.rb
class Product < ApplicationRecord
  belongs_to :category
  belongs_to :vendor, class_name: 'User'
  has_many :order_items
  has_many :reviews
  has_many_attached :images
  
  # Basic validations
  validates :name, presence: true, length: { minimum: 3, maximum: 200 }
  validates :description, presence: true, length: { minimum: 10, maximum: 5000 }
  validates :price, presence: true, numericality: { 
    greater_than: 0, 
    less_than: 1_000_000,
    precision: 2 
  }
  validates :sku, presence: true, uniqueness: true, 
            format: { with: /\A[A-Z0-9\-]+\z/, message: "only allows uppercase letters, numbers, and hyphens" }
  
  # Inventory validations
  validates :stock_quantity, presence: true, numericality: { 
    greater_than_or_equal_to: 0,
    only_integer: true
  }
  validates :reorder_level, numericality: { greater_than_or_equal_to: 0 }, allow_nil: true
  
  # Status and availability
  validates :status, inclusion: { in: %w[draft active inactive discontinued] }
  validates :availability, inclusion: { in: %w[in_stock out_of_stock pre_order] }
  
  # Dimensional validations
  validates :weight, numericality: { greater_than: 0 }, allow_nil: true
  validates :length, :width, :height, numericality: { greater_than: 0 }, 
            allow_nil: true, if: :physical_product?
  
  # Custom validations
  validate :price_must_be_reasonable_for_category
  validate :images_count_within_limit
  validate :discount_price_logic
  validate :availability_matches_stock
  
  # Conditional validations
  validates :digital_download_url, presence: true, if: :digital_product?
  validates :shipping_class, presence: true, if: :physical_product?
  validates :expiry_date, presence: true, if: :perishable?
  
  # Scoped validations
  validates :name, uniqueness: { scope: :vendor_id, 
                                message: "already exists for this vendor" }
  
  before_validation :generate_sku, on: :create, if: -> { sku.blank? }
  before_validation :set_availability_from_stock
  
  private
  
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
  
  def images_count_within_limit
    if images.attached? && images.count > 10
      errors.add(:images, "cannot exceed 10 images per product")
    end
    
    if images.attached? && images.count < 1
      errors.add(:images, "must have at least one product image")
    end
  end
  
  def discount_price_logic
    return unless discount_price && price
    
    if discount_price >= price
      errors.add(:discount_price, "must be less than regular price")
    end
    
    if discount_price < 0
      errors.add(:discount_price, "cannot be negative")
    end
    
    discount_percentage = ((price - discount_price) / price) * 100
    if discount_percentage > 90
      errors.add(:discount_price, "discount cannot exceed 90%")
    end
  end
  
  def availability_matches_stock
    if stock_quantity == 0 && availability == 'in_stock'
      errors.add(:availability, "cannot be 'in stock' when quantity is 0")
    end
    
    if stock_quantity > 0 && availability == 'out_of_stock'
      errors.add(:availability, "cannot be 'out of stock' when quantity is available")
    end
  end
  
  def physical_product?
    product_type == 'physical'
  end
  
  def digital_product?
    product_type == 'digital'
  end
  
  def perishable?
    category&.perishable?
  end
  
  def generate_sku
    self.sku = "#{category.code}-#{SecureRandom.alphanumeric(8).upcase}"
  end
  
  def set_availability_from_stock
    self.availability = stock_quantity > 0 ? 'in_stock' : 'out_of_stock'
  end
end
```

## Order Model

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :user
  belongs_to :shipping_address, class_name: 'Address'
  belongs_to :billing_address, class_name: 'Address'
  has_many :order_items, dependent: :destroy
  has_many :products, through: :order_items
  
  # Order number and status
  validates :order_number, presence: true, uniqueness: true
  validates :status, inclusion: { 
    in: %w[pending confirmed processing shipped delivered cancelled refunded] 
  }
  
  # Financial validations
  validates :subtotal, :tax_amount, :shipping_cost, :total_amount,
            presence: true, numericality: { greater_than_or_equal_to: 0 }
  validates :discount_amount, numericality: { greater_than_or_equal_to: 0 }, 
            allow_nil: true
  
  # Payment validations
  validates :payment_method, inclusion: { 
    in: %w[credit_card debit_card paypal bank_transfer cash_on_delivery] 
  }
  validates :payment_status, inclusion: { 
    in: %w[pending paid failed refunded partially_refunded] 
  }
  
  # Date validations
  validates :order_date, presence: true
  validate :order_date_not_in_future
  validate :delivery_date_after_order_date
  
  # Custom business logic validations
  validate :total_calculation_is_correct
  validate :order_items_present
  validate :shipping_address_deliverable
  validate :payment_method_available_for_total
  validate :user_can_place_order
  
  # Conditional validations
  validates :tracking_number, presence: true, if: :shipped_or_delivered?
  validates :delivery_date, presence: true, if: :delivered?
  validates :cancellation_reason, presence: true, if: :cancelled?
  validates :refund_amount, presence: true, numericality: { greater_than: 0 }, 
            if: :refunded?
  
  before_validation :generate_order_number, on: :create
  before_validation :calculate_totals
  
  private
  
  def order_date_not_in_future
    if order_date && order_date > Date.current
      errors.add(:order_date, "cannot be in the future")
    end
  end
  
  def delivery_date_after_order_date
    if delivery_date && order_date && delivery_date < order_date
      errors.add(:delivery_date, "must be after order date")
    end
  end
  
  def total_calculation_is_correct
    return unless subtotal && tax_amount && shipping_cost && total_amount
    
    calculated_total = subtotal + tax_amount + shipping_cost
    calculated_total -= discount_amount if discount_amount
    
    unless (calculated_total - total_amount).abs < 0.01
      errors.add(:total_amount, "does not match calculated total")
    end
  end
  
  def order_items_present
    if order_items.empty?
      errors.add(:base, "Order must contain at least one item")
    end
  end
  
  def shipping_address_deliverable
    return unless shipping_address
    
    unless shipping_address.deliverable?
      errors.add(:shipping_address, "is not in a deliverable area")
    end
  end
  
  def payment_method_available_for_total
    if payment_method == 'cash_on_delivery' && total_amount > 1000
      errors.add(:payment_method, "Cash on delivery not available for orders over $1000")
    end
  end
  
  def user_can_place_order
    if user.suspended?
      errors.add(:user, "account is suspended and cannot place orders")
    end
    
    if user.orders.where(payment_status: 'failed').count > 3
      errors.add(:user, "has too many failed payment attempts")
    end
  end
  
  def shipped_or_delivered?
    %w[shipped delivered].include?(status)
  end
  
  def delivered?
    status == 'delivered'
  end
  
  def cancelled?
    status == 'cancelled'
  end
  
  def refunded?
    status == 'refunded'
  end
  
  def generate_order_number
    self.order_number = "ORD-#{Date.current.strftime('%Y%m%d')}-#{SecureRandom.hex(4).upcase}"
  end
  
  def calculate_totals
    return unless order_items.any?
    
    self.subtotal = order_items.sum(&:total_price)
    self.tax_amount = subtotal * 0.08 # 8% tax rate
    self.shipping_cost = calculate_shipping_cost
    self.total_amount = subtotal + tax_amount + shipping_cost - (discount_amount || 0)
  end
  
  def calculate_shipping_cost
    # Simplified shipping calculation
    return 0 if subtotal > 100 # Free shipping over $100
    return 15 # Standard shipping
  end
end
```

## Review Model

```ruby
# app/models/review.rb
class Review < ApplicationRecord
  belongs_to :user
  belongs_to :product
  
  validates :rating, presence: true, inclusion: { in: 1..5 }
  validates :title, presence: true, length: { minimum: 5, maximum: 100 }
  validates :content, presence: true, length: { minimum: 10, maximum: 2000 }
  
  # Prevent duplicate reviews
  validates :user_id, uniqueness: { scope: :product_id, 
                                   message: "can only review each product once" }
  
  # Business logic validations
  validate :user_purchased_product
  validate :no_profanity_in_content
  validate :review_not_too_soon_after_purchase
  
  before_validation :strip_whitespace
  
  private
  
  def user_purchased_product
    return if user.admin? # Admins can review without purchasing
    
    unless user.orders.joins(:order_items)
                     .where(order_items: { product_id: product_id })
                     .where(status: 'delivered').exists?
      errors.add(:base, "You can only review products you have purchased")
    end
  end
  
  def no_profanity_in_content
    profanity_words = %w[badword1 badword2] # Would use a proper profanity filter
    
    if profanity_words.any? { |word| content.downcase.include?(word.downcase) }
      errors.add(:content, "contains inappropriate language")
    end
  end
  
  def review_not_too_soon_after_purchase
    return unless user_purchased_product
    
    latest_purchase = user.orders.joins(:order_items)
                         .where(order_items: { product_id: product_id })
                         .where(status: 'delivered')
                         .order(:delivery_date)
                         .last
    
    if latest_purchase && latest_purchase.delivery_date > 3.days.ago
      errors.add(:base, "Please wait at least 3 days after delivery before reviewing")
    end
  end
  
  def strip_whitespace
    self.title = title.strip if title
    self.content = content.strip if content
  end
end
```

## Custom Validator Class

```ruby
# Custom validator class for reusable validation logic
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
# validates :email, email: true
```

## Additional Model Examples

### Address Model

```ruby
# app/models/address.rb
class Address < ApplicationRecord
  belongs_to :user
  
  validates :street_address, presence: true, length: { minimum: 5, maximum: 100 }
  validates :city, presence: true, length: { minimum: 2, maximum: 50 }
  validates :state, presence: true, length: { is: 2 }
  validates :postal_code, presence: true, format: { with: /\A\d{5}(-\d{4})?\z/ }
  validates :country, presence: true, inclusion: { in: ISO3166::Country.codes }
  
  validates :address_type, inclusion: { in: %w[home work billing shipping] }
  
  # Only one default address per type per user
  validates :default, uniqueness: { scope: [:user_id, :address_type], 
                                   message: "can only have one default address per type" },
                     if: :default?
  
  validate :deliverable_address
  
  before_validation :normalize_postal_code
  
  private
  
  def deliverable_address
    # Check if address is in deliverable area
    undeliverable_states = %w[AK HI] # Example: Alaska and Hawaii
    if undeliverable_states.include?(state)
      errors.add(:state, "is not in our delivery area")
    end
  end
  
  def normalize_postal_code
    self.postal_code = postal_code.gsub(/\D/, '') if postal_code.present?
  end
  
  def deliverable?
    !%w[AK HI].include?(state)
  end
end
```

### Category Model

```ruby
# app/models/category.rb
class Category < ApplicationRecord
  has_many :products
  belongs_to :parent_category, class_name: 'Category', optional: true
  has_many :subcategories, class_name: 'Category', foreign_key: 'parent_category_id'
  
  validates :name, presence: true, uniqueness: { scope: :parent_category_id }
  validates :code, presence: true, uniqueness: true, 
            format: { with: /\A[A-Z0-9_]+\z/, message: "only uppercase letters, numbers and underscores" }
  validates :description, length: { maximum: 500 }
  validates :sort_order, numericality: { only_integer: true, greater_than_or_equal_to: 0 }
  
  validate :parent_category_cannot_be_self
  validate :category_depth_limit
  
  scope :active, -> { where(active: true) }
  scope :root_categories, -> { where(parent_category_id: nil) }
  
  private
  
  def parent_category_cannot_be_self
    if parent_category_id == id
      errors.add(:parent_category, "cannot be the same as this category")
    end
  end
  
  def category_depth_limit
    depth = 0
    current_category = parent_category
    
    while current_category && depth < 5
      depth += 1
      current_category = current_category.parent_category
    end
    
    if depth >= 4
      errors.add(:parent_category, "category nesting too deep (maximum 4 levels)")
    end
  end
  
  def perishable?
    %w[FOOD_FRESH FOOD_FROZEN].include?(code)
  end
end
```

## Testing Examples

### RSpec Model Tests

```ruby
# spec/models/user_spec.rb
RSpec.describe User, type: :model do
  describe 'validations' do
    subject { build(:user) }
    
    it { should validate_presence_of(:first_name) }
    it { should validate_presence_of(:last_name) }
    it { should validate_presence_of(:email) }
    it { should validate_uniqueness_of(:email).case_insensitive }
    
    describe 'password validations' do
      it 'requires minimum 8 characters' do
        user = build(:user, password: '1234567')
        expect(user).not_to be_valid
        expect(user.errors[:password]).to include('is too short (minimum is 8 characters)')
      end
      
      it 'requires complex password format' do
        user = build(:user, password: 'password123')
        expect(user).not_to be_valid
        expect(user.errors[:password]).to include('must include uppercase, lowercase, number and special character')
      end
      
      it 'accepts valid complex password' do
        user = build(:user, password: 'Password123!')
        expect(user).to be_valid
      end
    end
    
    describe 'age validation' do
      it 'rejects users under 13' do
        user = build(:user, date_of_birth: 10.years.ago)
        expect(user).not_to be_valid
        expect(user.errors[:date_of_birth]).to include('You must be at least 13 years old')
      end
      
      it 'accepts users 13 and older' do
        user = build(:user, date_of_birth: 15.years.ago)
        expect(user).to be_valid
      end
      
      it 'rejects unrealistic birth dates' do
        user = build(:user, date_of_birth: 150.years.ago)
        expect(user).not_to be_valid
        expect(user.errors[:date_of_birth]).to include('Birth date seems unrealistic')
      end
    end
    
    describe 'business account validations' do
      it 'requires company name for business accounts' do
        user = build(:user, account_type: 'business', company_name: nil)
        expect(user).not_to be_valid
        expect(user.errors[:company_name]).to include("can't be blank")
      end
      
      it 'requires tax ID for business accounts' do
        user = build(:user, account_type: 'business', tax_id: nil)
        expect(user).not_to be_valid
        expect(user.errors[:tax_id]).to include("can't be blank")
      end
      
      it 'validates tax ID format' do
        user = build(:user, account_type: 'business', tax_id: '123456789')
        expect(user).not_to be_valid
        expect(user.errors[:tax_id]).to include('is invalid')
      end
    end
  end
  
  describe 'data normalization' do
    it 'normalizes email to lowercase' do
      user = create(:user, email: 'TEST@EXAMPLE.COM')
      expect(user.email).to eq('test@example.com')
    end
    
    it 'strips phone number of non-digits' do
      user = create(:user, phone: '(555) 123-4567')
      expect(user.phone).to eq('5551234567')
    end
  end
end

# spec/models/product_spec.rb
RSpec.describe Product, type: :model do
  describe 'validations' do
    subject { build(:product) }
    
    it { should validate_presence_of(:name) }
    it { should validate_presence_of(:price) }
    it { should validate_uniqueness_of(:sku) }
    
    describe 'price validations' do
      it 'rejects negative prices' do
        product = build(:product, price: -10)
        expect(product).not_to be_valid
        expect(product.errors[:price]).to include('must be greater than 0')
      end
      
      it 'rejects extremely high prices' do
        product = build(:product, price: 2_000_000)
        expect(product).not_to be_valid
        expect(product.errors[:price]).to include('must be less than 1000000')
      end
    end
    
    describe 'category-specific price validation' do
      it 'validates reasonable prices for electronics' do
        electronics = create(:category, name: 'Electronics')
        product = build(:product, category: electronics, price: 100_000)
        expect(product).not_to be_valid
        expect(product.errors[:price]).to include('seems too high for electronics category')
      end
      
      it 'validates reasonable prices for books' do
        books = create(:category, name: 'Books')
        product = build(:product, category: books, price: 1000)
        expect(product).not_to be_valid
        expect(product.errors[:price]).to include('seems too high for books category')
      end
    end
    
    describe 'discount validation' do
      it 'ensures discount price is less than regular price' do
        product = build(:product, price: 100, discount_price: 120)
        expect(product).not_to be_valid
        expect(product.errors[:discount_price]).to include('must be less than regular price')
      end
      
      it 'prevents excessive discounts' do
        product = build(:product, price: 100, discount_price: 5)
        expect(product).not_to be_valid
        expect(product.errors[:discount_price]).to include('discount cannot exceed 90%')
      end
    end
  end
end

# spec/models/order_spec.rb
RSpec.describe Order, type: :model do
  describe 'validations' do
    subject { build(:order) }
    
    it { should validate_presence_of(:order_number) }
    it { should validate_uniqueness_of(:order_number) }
    
    describe 'financial calculations' do
      it 'validates total calculation accuracy' do
        order = build(:order, 
                     subtotal: 100, 
                     tax_amount: 8, 
                     shipping_cost: 10, 
                     total_amount: 999)
        expect(order).not_to be_valid
        expect(order.errors[:total_amount]).to include('does not match calculated total')
      end
      
      it 'accepts correct total calculations' do
        order = build(:order, 
                     subtotal: 100, 
                     tax_amount: 8, 
                     shipping_cost: 10, 
                     total_amount: 118)
        expect(order).to be_valid
      end
    end
    
    describe 'business logic validations' do
      it 'prevents orders from suspended users' do
        suspended_user = create(:user, status: 'suspended')
        order = build(:order, user: suspended_user)
        expect(order).not_to be_valid
        expect(order.errors[:user]).to include('account is suspended and cannot place orders')
      end
      
      it 'restricts cash on delivery for high-value orders' do
        order = build(:order, payment_method: 'cash_on_delivery', total_amount: 1500)
        expect(order).not_to be_valid
        expect(order.errors[:payment_method]).to include('Cash on delivery not available for orders over $1000')
      end
    end
  end
end
```

This markdown file contains all the complete Rails validation code examples from the previous artifact, formatted as a single reference document with all the models and testing examples included.