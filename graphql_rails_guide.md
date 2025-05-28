# Complete GraphQL with Ruby on Rails Guide

A comprehensive guide to implementing GraphQL in Ruby on Rails applications, covering everything from basic setup to advanced production patterns.

## Table of Contents

1. [Introduction and Setup](#introduction-and-setup)
2. [Schema Definition](#schema-definition)
3. [Types and Fields](#types-and-fields)
4. [Queries](#queries)
5. [Mutations](#mutations)
6. [Subscriptions](#subscriptions)
7. [Authentication and Authorization](#authentication-and-authorization)
8. [Error Handling](#error-handling)
9. [Performance Optimization](#performance-optimization)
10. [Testing](#testing)
11. [Production Considerations](#production-considerations)

## Introduction and Setup

GraphQL is a query language and runtime for APIs that provides a more efficient, powerful alternative to REST. It allows clients to request exactly the data they need.

### Installation

Add the GraphQL gem to your Gemfile:

```ruby
# Gemfile
gem 'graphql'
gem 'graphiql-rails', group: :development
gem 'graphql-batch' # For batching and performance
```

### Initial Setup

```bash
bundle install
rails generate graphql:install
```

This creates:
- `app/graphql/` directory structure
- `app/graphql/types/` for type definitions
- `app/graphql/mutations/` for mutations
- `app/controllers/graphql_controller.rb`
- GraphiQL interface at `/graphiql`

### Basic Configuration

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  mutation(Types::MutationType)
  query(Types::QueryType)
  subscription(Types::SubscriptionType)
  
  # Enable batch loading
  use GraphQL::Batch
  
  # Add built-in connections for pagination
  use GraphQL::Pagination::Connections
  
  # Rescue from errors
  rescue_from(ActiveRecord::RecordNotFound) do |error|
    GraphQL::ExecutionError.new("Record not found")
  end
end
```

## Schema Definition

### Models Setup

Let's use an e-commerce example with these models:

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :orders, dependent: :destroy
  has_many :reviews, dependent: :destroy
  has_one :cart, dependent: :destroy
  
  enum role: { customer: 0, admin: 1, vendor: 2 }
  enum status: { active: 0, inactive: 1, suspended: 2 }
  
  validates :email, presence: true, uniqueness: true
  validates :first_name, :last_name, presence: true
end

# app/models/product.rb
class Product < ApplicationRecord
  belongs_to :category
  belongs_to :vendor, class_name: 'User'
  has_many :order_items
  has_many :reviews
  has_many_attached :images
  
  enum status: { draft: 0, active: 1, inactive: 2, discontinued: 3 }
  
  validates :name, :description, :price, presence: true
  validates :price, numericality: { greater_than: 0 }
  validates :sku, presence: true, uniqueness: true
end

# app/models/order.rb
class Order < ApplicationRecord
  belongs_to :user
  has_many :order_items, dependent: :destroy
  has_many :products, through: :order_items
  
  enum status: { pending: 0, confirmed: 1, shipped: 2, delivered: 3, cancelled: 4 }
  
  validates :order_number, presence: true, uniqueness: true
  validates :total_amount, presence: true, numericality: { greater_than: 0 }
end

# app/models/category.rb
class Category < ApplicationRecord
  has_many :products
  belongs_to :parent_category, class_name: 'Category', optional: true
  has_many :subcategories, class_name: 'Category', foreign_key: 'parent_category_id'
  
  validates :name, presence: true, uniqueness: { scope: :parent_category_id }
end
```

## Types and Fields

### Base Types

```ruby
# app/graphql/types/base_object.rb
module Types
  class BaseObject < GraphQL::Schema::Object
    edge_type_class(Types::BaseEdge)
    connection_type_class(Types::BaseConnection)
    field_class Types::BaseField
  end
end

# app/graphql/types/base_field.rb
module Types
  class BaseField < GraphQL::Schema::Field
    argument_class Types::BaseArgument
    
    def resolve_field(obj, args, ctx)
      # Add field-level authorization
      authorize!(obj, ctx)
      super
    end
    
    private
    
    def authorize!(obj, ctx)
      # Override in specific fields for authorization
    end
  end
end
```

### User Type

```ruby
# app/graphql/types/user_type.rb
module Types
  class UserType < Types::BaseObject
    description "A user in the system"
    
    field :id, ID, null: false
    field :email, String, null: false
    field :first_name, String, null: false
    field :last_name, String, null: false
    field :full_name, String, null: false
    field :role, String, null: false
    field :status, String, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
    
    # Associations
    field :orders, [Types::OrderType], null: false
    field :reviews, [Types::ReviewType], null: false
    field :cart, Types::CartType, null: true
    
    # Connection fields for pagination
    field :orders_connection, Types::OrderType.connection_type, null: false do
      argument :status, String, required: false
      argument :date_from, GraphQL::Types::ISO8601Date, required: false
      argument :date_to, GraphQL::Types::ISO8601Date, required: false
    end
    
    # Computed fields
    field :orders_count, Integer, null: false
    field :total_spent, Float, null: false
    field :average_order_value, Float, null: false
    
    # Conditional fields based on permissions
    field :phone, String, null: true do
      description "Phone number (visible to user and admins only)"
    end
    
    def full_name
      "#{object.first_name} #{object.last_name}"
    end
    
    def orders_connection(status: nil, date_from: nil, date_to: nil)
      scope = object.orders
      scope = scope.where(status: status) if status
      scope = scope.where(created_at: date_from..) if date_from
      scope = scope.where(created_at: ..date_to) if date_to
      scope
    end
    
    def orders_count
      BatchLoader::GraphQL.for(object.id).batch do |user_ids, loader|
        counts = Order.where(user_id: user_ids).group(:user_id).count
        user_ids.each { |id| loader.call(id, counts[id] || 0) }
      end
    end
    
    def total_spent
      BatchLoader::GraphQL.for(object.id).batch do |user_ids, loader|
        totals = Order.where(user_id: user_ids, status: :delivered)
                     .group(:user_id)
                     .sum(:total_amount)
        user_ids.each { |id| loader.call(id, totals[id] || 0.0) }
      end
    end
    
    def average_order_value
      BatchLoader::GraphQL.for(object.id).batch do |user_ids, loader|
        averages = Order.where(user_id: user_ids, status: :delivered)
                       .group(:user_id)
                       .average(:total_amount)
        user_ids.each { |id| loader.call(id, averages[id] || 0.0) }
      end
    end
    
    def phone
      # Only show phone to the user themselves or admins
      return object.phone if context[:current_user]&.admin?
      return object.phone if context[:current_user] == object
      nil
    end
  end
end
```

### Product Type

```ruby
# app/graphql/types/product_type.rb
module Types
  class ProductType < Types::BaseObject
    description "A product in the catalog"
    
    field :id, ID, null: false
    field :name, String, null: false
    field :description, String, null: false
    field :sku, String, null: false
    field :price, Float, null: false
    field :discount_price, Float, null: true
    field :stock_quantity, Integer, null: false
    field :status, String, null: false
    field :weight, Float, null: true
    field :dimensions, Types::DimensionsType, null: true
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
    
    # Associations
    field :category, Types::CategoryType, null: false
    field :vendor, Types::UserType, null: false
    field :reviews, [Types::ReviewType], null: false
    field :images, [String], null: false
    
    # Computed fields
    field :average_rating, Float, null: true
    field :reviews_count, Integer, null: false
    field :is_in_stock, Boolean, null: false
    field :effective_price, Float, null: false
    field :discount_percentage, Float, null: true
    
    # Search and filtering
    field :related_products, [Types::ProductType], null: false do
      argument :limit, Integer, required: false, default_value: 5
    end
    
    def images
      object.images.attached? ? object.images.map { |img| Rails.application.routes.url_helpers.url_for(img) } : []
    end
    
    def dimensions
      return nil unless object.length && object.width && object.height
      {
        length: object.length,
        width: object.width,
        height: object.height
      }
    end
    
    def average_rating
      BatchLoader::GraphQL.for(object.id).batch do |product_ids, loader|
        averages = Review.where(product_id: product_ids)
                        .group(:product_id)
                        .average(:rating)
        product_ids.each { |id| loader.call(id, averages[id]&.round(2)) }
      end
    end
    
    def reviews_count
      BatchLoader::GraphQL.for(object.id).batch do |product_ids, loader|
        counts = Review.where(product_id: product_ids).group(:product_id).count
        product_ids.each { |id| loader.call(id, counts[id] || 0) }
      end
    end
    
    def is_in_stock
      object.stock_quantity > 0
    end
    
    def effective_price
      object.discount_price || object.price
    end
    
    def discount_percentage
      return nil unless object.discount_price
      ((object.price - object.discount_price) / object.price * 100).round(2)
    end
    
    def related_products(limit:)
      Product.joins(:category)
             .where(category: object.category)
             .where.not(id: object.id)
             .where(status: :active)
             .limit(limit)
    end
  end
end

# app/graphql/types/dimensions_type.rb
module Types
  class DimensionsType < Types::BaseObject
    field :length, Float, null: false
    field :width, Float, null: false
    field :height, Float, null: false
  end
end
```

### Order Type

```ruby
# app/graphql/types/order_type.rb
module Types
  class OrderType < Types::BaseObject
    description "An order placed by a user"
    
    field :id, ID, null: false
    field :order_number, String, null: false
    field :status, String, null: false
    field :subtotal, Float, null: false
    field :tax_amount, Float, null: false
    field :shipping_cost, Float, null: false
    field :discount_amount, Float, null: true
    field :total_amount, Float, null: false
    field :order_date, GraphQL::Types::ISO8601DateTime, null: false
    field :shipped_date, GraphQL::Types::ISO8601DateTime, null: true
    field :delivered_date, GraphQL::Types::ISO8601DateTime, null: true
    field :tracking_number, String, null: true
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
    
    # Associations
    field :user, Types::UserType, null: false
    field :order_items, [Types::OrderItemType], null: false
    field :products, [Types::ProductType], null: false
    
    # Computed fields
    field :items_count, Integer, null: false
    field :can_be_cancelled, Boolean, null: false
    field :estimated_delivery, GraphQL::Types::ISO8601Date, null: true
    
    def items_count
      object.order_items.sum(:quantity)
    end
    
    def can_be_cancelled
      %w[pending confirmed].include?(object.status)
    end
    
    def estimated_delivery
      return nil unless object.shipped_date
      object.shipped_date.to_date + 7.days
    end
  end
end

# app/graphql/types/order_item_type.rb
module Types
  class OrderItemType < Types::BaseObject
    field :id, ID, null: false
    field :quantity, Integer, null: false
    field :unit_price, Float, null: false
    field :total_price, Float, null: false
    field :product, Types::ProductType, null: false
    field :order, Types::OrderType, null: false
  end
end
```

### Category Type

```ruby
# app/graphql/types/category_type.rb
module Types
  class CategoryType < Types::BaseObject
    description "A product category"
    
    field :id, ID, null: false
    field :name, String, null: false
    field :description, String, null: true
    field :slug, String, null: false
    field :sort_order, Integer, null: false
    field :active, Boolean, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    
    # Associations
    field :products, [Types::ProductType], null: false
    field :products_connection, Types::ProductType.connection_type, null: false
    field :parent_category, Types::CategoryType, null: true
    field :subcategories, [Types::CategoryType], null: false
    
    # Computed fields
    field :products_count, Integer, null: false
    field :is_root_category, Boolean, null: false
    field :breadcrumb, [Types::CategoryType], null: false
    
    def products_connection
      object.products.active
    end
    
    def products_count
      BatchLoader::GraphQL.for(object.id).batch do |category_ids, loader|
        counts = Product.where(category_id: category_ids, status: :active)
                       .group(:category_id)
                       .count
        category_ids.each { |id| loader.call(id, counts[id] || 0) }
      end
    end
    
    def is_root_category
      object.parent_category_id.nil?
    end
    
    def breadcrumb
      breadcrumb = []
      current = object
      while current
        breadcrumb.unshift(current)
        current = current.parent_category
      end
      breadcrumb
    end
  end
end
```

## Queries

### Query Root Type

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    description "The query root"
    
    # User queries
    field :current_user, Types::UserType, null: true do
      description "Get the currently authenticated user"
    end
    
    field :user, Types::UserType, null: true do
      description "Find a user by ID"
      argument :id, ID, required: true
    end
    
    field :users, Types::UserType.connection_type, null: false do
      description "List all users (admin only)"
      argument :role, String, required: false
      argument :status, String, required: false
      argument :search, String, required: false
    end
    
    # Product queries
    field :product, Types::ProductType, null: true do
      description "Find a product by ID or SKU"
      argument :id, ID, required: false
      argument :sku, String, required: false
    end
    
    field :products, Types::ProductType.connection_type, null: false do
      description "List products with filtering and search"
      argument :category_id, ID, required: false
      argument :vendor_id, ID, required: false
      argument :status, String, required: false
      argument :search, String, required: false
      argument :price_min, Float, required: false
      argument :price_max, Float, required: false
      argument :in_stock_only, Boolean, required: false, default_value: false
    end
    
    field :featured_products, [Types::ProductType], null: false do
      description "Get featured products"
      argument :limit, Integer, required: false, default_value: 10
    end
    
    # Category queries
    field :category, Types::CategoryType, null: true do
      description "Find a category by ID"
      argument :id, ID, required: true
    end
    
    field :categories, [Types::CategoryType], null: false do
      description "List all categories"
      argument :parent_id, ID, required: false
      argument :active_only, Boolean, required: false, default_value: true
    end
    
    field :root_categories, [Types::CategoryType], null: false do
      description "Get root level categories"
    end
    
    # Order queries
    field :order, Types::OrderType, null: true do
      description "Find an order by ID"
      argument :id, ID, required: true
    end
    
    field :my_orders, Types::OrderType.connection_type, null: false do
      description "Get current user's orders"
      argument :status, String, required: false
    end
    
    # Search queries
    field :search, Types::SearchResultType, null: false do
      description "Global search across products and categories"
      argument :query, String, required: true
      argument :limit, Integer, required: false, default_value: 20
    end
    
    # Analytics queries (admin only)
    field :sales_analytics, Types::SalesAnalyticsType, null: false do
      description "Get sales analytics data"
      argument :date_from, GraphQL::Types::ISO8601Date, required: false
      argument :date_to, GraphQL::Types::ISO8601Date, required: false
    end
    
    # Resolvers
    def current_user
      context[:current_user]
    end
    
    def user(id:)
      authorize_admin!
      User.find(id)
    end
    
    def users(role: nil, status: nil, search: nil)
      authorize_admin!
      
      scope = User.all
      scope = scope.where(role: role) if role
      scope = scope.where(status: status) if status
      
      if search
        scope = scope.where("first_name ILIKE ? OR last_name ILIKE ? OR email ILIKE ?", 
                           "%#{search}%", "%#{search}%", "%#{search}%")
      end
      
      scope
    end
    
    def product(id: nil, sku: nil)
      return Product.find(id) if id
      return Product.find_by(sku: sku) if sku
      raise GraphQL::ExecutionError, "Must provide either id or sku"
    end
    
    def products(category_id: nil, vendor_id: nil, status: nil, search: nil, 
                price_min: nil, price_max: nil, in_stock_only: false)
      scope = Product.includes(:category, :vendor)
      
      scope = scope.where(category_id: category_id) if category_id
      scope = scope.where(vendor_id: vendor_id) if vendor_id
      scope = scope.where(status: status) if status
      scope = scope.where("stock_quantity > 0") if in_stock_only
      scope = scope.where("price >= ?", price_min) if price_min
      scope = scope.where("price <= ?", price_max) if price_max
      
      if search
        scope = scope.where("name ILIKE ? OR description ILIKE ? OR sku ILIKE ?",
                           "%#{search}%", "%#{search}%", "%#{search}%")
      end
      
      scope
    end
    
    def featured_products(limit:)
      Product.where(featured: true, status: :active).limit(limit)
    end
    
    def category(id:)
      Category.find(id)
    end
    
    def categories(parent_id: nil, active_only: true)
      scope = Category.all
      scope = scope.where(parent_category_id: parent_id) if parent_id
      scope = scope.where(active: true) if active_only
      scope.order(:sort_order, :name)
    end
    
    def root_categories
      Category.where(parent_category_id: nil, active: true).order(:sort_order, :name)
    end
    
    def order(id:)
      order = Order.find(id)
      
      # Users can only see their own orders, admins can see all
      unless context[:current_user]&.admin? || order.user == context[:current_user]
        raise GraphQL::ExecutionError, "Access denied"
      end
      
      order
    end
    
    def my_orders(status: nil)
      raise GraphQL::ExecutionError, "Authentication required" unless context[:current_user]
      
      scope = context[:current_user].orders
      scope = scope.where(status: status) if status
      scope.order(created_at: :desc)
    end
    
    def search(query:, limit:)
      # Implement search across products and categories
      products = Product.where(status: :active)
                       .where("name ILIKE ? OR description ILIKE ?", "%#{query}%", "%#{query}%")
                       .limit(limit / 2)
      
      categories = Category.where(active: true)
                          .where("name ILIKE ? OR description ILIKE ?", "%#{query}%", "%#{query}%")
                          .limit(limit / 2)
      
      {
        products: products,
        categories: categories,
        total_count: products.count + categories.count
      }
    end
    
    def sales_analytics(date_from: nil, date_to: nil)
      authorize_admin!
      
      scope = Order.where(status: :delivered)
      scope = scope.where(created_at: date_from..) if date_from
      scope = scope.where(created_at: ..date_to) if date_to
      
      {
        total_revenue: scope.sum(:total_amount),
        total_orders: scope.count,
        average_order_value: scope.average(:total_amount),
        top_products: scope.joins(:order_items)
                          .joins("JOIN products ON order_items.product_id = products.id")
                          .group("products.id, products.name")
                          .order("SUM(order_items.quantity) DESC")
                          .limit(10)
      }
    end
    
    private
    
    def authorize_admin!
      unless context[:current_user]&.admin?
        raise GraphQL::ExecutionError, "Admin access required"
      end
    end
  end
end
```

### Search Result Type

```ruby
# app/graphql/types/search_result_type.rb
module Types
  class SearchResultType < Types::BaseObject
    field :products, [Types::ProductType], null: false
    field :categories, [Types::CategoryType], null: false
    field :total_count, Integer, null: false
  end
end
```

### Sales Analytics Type

```ruby
# app/graphql/types/sales_analytics_type.rb
module Types
  class SalesAnalyticsType < Types::BaseObject
    field :total_revenue, Float, null: false
    field :total_orders, Integer, null: false
    field :average_order_value, Float, null: false
    field :top_products, [Types::ProductSalesType], null: false
  end
  
  class ProductSalesType < Types::BaseObject
    field :product, Types::ProductType, null: false
    field :quantity_sold, Integer, null: false
    field :revenue, Float, null: false
  end
end
```

## Mutations

### Base Mutation

```ruby
# app/graphql/mutations/base_mutation.rb
module Mutations
  class BaseMutation < GraphQL::Schema::RelayClassicMutation
    argument_class Types::BaseArgument
    field_class Types::BaseField
    input_object_class Types::BaseInputObject
    object_class Types::BaseObject
    
    protected
    
    def current_user
      context[:current_user]
    end
    
    def authenticate_user!
      raise GraphQL::ExecutionError, "Authentication required" unless current_user
    end
    
    def authorize_admin!
      authenticate_user!
      unless current_user.admin?
        raise GraphQL::ExecutionError, "Admin access required"
      end
    end
    
    def authorize_owner!(resource)
      authenticate_user!
      unless current_user.admin? || resource.user == current_user
        raise GraphQL::ExecutionError, "Access denied"
      end
    end
  end
end
```

### User Mutations

```ruby
# app/graphql/mutations/user/create_user.rb
module Mutations
  module User
    class CreateUser < BaseMutation
      description "Create a new user account"
      
      argument :email, String, required: true
      argument :password, String, required: true
      argument :first_name, String, required: true
      argument :last_name, String, required: true
      argument :phone, String, required: false
      
      field :user, Types::UserType, null: true
      field :errors, [String], null: false
      
      def resolve(email:, password:, first_name:, last_name:, phone: nil)
        user = User.new(
          email: email,
          password: password,
          first_name: first_name,
          last_name: last_name,
          phone: phone
        )
        
        if user.save
          # Send welcome email
          UserMailer.welcome_email(user).deliver_later
          
          {
            user: user,
            errors: []
          }
        else
          {
            user: nil,
            errors: user.errors.full_messages
          }
        end
      end
    end
  end
end

# app/graphql/mutations/user/update_user.rb
module Mutations
  module User
    class UpdateUser < BaseMutation
      description "Update user profile"
      
      argument :id, ID, required: true
      argument :first_name, String, required: false
      argument :last_name, String, required: false
      argument :phone, String, required: false
      argument :email, String, required: false
      
      field :user, Types::UserType, null: true
      field :errors, [String], null: false
      
      def resolve(id:, **attributes)
        user = User.find(id)
        authorize_owner!(user)
        
        if user.update(attributes.compact)
          {
            user: user,
            errors: []
          }
        else
          {
            user: nil,
            errors: user.errors.full_messages
          }
        end
      end
    end
  end
end

# app/graphql/mutations/user/delete_user.rb
module Mutations
  module User
    class DeleteUser < BaseMutation
      description "Delete user account"
      
      argument :id, ID, required: true
      
      field :success, Boolean, null: false
      field :errors, [String], null: false
      
      def resolve(id:)
        user = User.find(id)
        authorize_owner!(user)
        
        if user.destroy
          {
            success: true,
            errors: []
          }
        else
          {
            success: false,
            errors: user.errors.full_messages
          }
        end
      end
    end
  end
end
```

### Product Mutations

```ruby
# app/graphql/mutations/product/create_product.rb
module Mutations
  module Product
    class CreateProduct < BaseMutation
      description "Create a new product"
      
      argument :name, String, required: true
      argument :description, String, required: true
      argument :price, Float, required: true
      argument :category_id, ID, required: true
      argument :sku, String, required: false
      argument :stock_quantity, Integer, required: true
      argument :weight, Float, required: false
      argument :images, [String], required: false
      
      field :product, Types::ProductType, null: true
      field :errors, [String], null: false
      
      def resolve(name:, description:, price:, category_id:, stock_quantity:, 
                  sku: nil, weight: nil, images: [])
        authenticate_user!
        
        product = Product.new(
          name: name,
          description: description,
          price: price,
          category_id: category_id,
          vendor: current_user,
          sku: sku || generate_sku,
          stock_quantity: stock_quantity,
          weight: weight,
          status: :draft
        )
        
        if product.save
          # Handle image uploads if provided
          attach_images(product, images) if images.any?
          
          {
            product: product,
            errors: []
          }
        else
          {
            product: nil,
            errors: product.errors.full_messages
          }
        end
      end
      
      private
      
      def generate_sku
        "#{Time.current.strftime('%Y%m%d')}-#{SecureRandom.hex(4).upcase}"
      end
      
      def attach_images(product, image_urls)
        # In a real app, you'd handle file uploads differently
        # This is a simplified example
        image_urls.each do |url|
          # Download and attach image
          # product.images.attach(...)
        end
      end
    end
  end
end

# app/graphql/mutations/product/update_product.rb
module Mutations
  module Product
    class UpdateProduct < BaseMutation
      description "Update a product"
      
      argument :id, ID, required: true
      argument :name, String, required: false
      argument :description, String, required: false
      argument :price, Float, required: false
      argument :discount_price, Float, required: false
      argument :stock_quantity, Integer, required: false
      argument :status, String, required: false
      
      field :product, Types::ProductType, null: true
      field :errors, [String], null: false
      
      def resolve(id:, **attributes)
        product = Product.find(id)
        
        # Only vendor or admin can update
        unless current_user&.admin? || product.vendor == current_user
          raise GraphQL::ExecutionError, "Access denied"
        end
        
        if product.update(attributes.compact)
          {
            product: product,
            errors: []
          }
        else
          {
            product: nil,
            errors: product.errors.full_messages
          }
        end
      end
    end
  end
end

# app/graphql/mutations/product/delete_product.rb
module Mutations
  module Product
    class DeleteProduct < BaseMutation
      description "Delete a product"
      
      argument :id, ID, required: true
      
      field :success, Boolean, null: false
      field :errors, [String], null: false
      
      def resolve(id:)
        product = Product.find(id)
        
        unless current_user&.admin? || product.vendor == current_user
          raise GraphQL::ExecutionError, "Access denied"
        end
        
        if product.destroy
          {
            success: true,
            errors: []
          }
        else
          {
            success: false,
            errors: product.errors.full_messages
          }
        end
      end
    end
  end
end
```

### Order Mutations

```ruby
# app/graphql/mutations/order/create_order.rb
module Mutations
  module Order
    class CreateOrder < BaseMutation
      description "Create a new order"
      
      argument :items, [Types::OrderItemInputType], required: true
      argument :shipping_address_id, ID, required: true
      argument :payment_method, String, required: true
      
      field :order, Types::OrderType, null: true
      field :errors, [String], null: false
      
      def resolve(items:, shipping_address_id:, payment_method:)
        authenticate_user!
        
        ActiveRecord::Base.transaction do
          order = Order.new(
            user: current_user,
            order_number: generate_order_number,
            shipping_address_id: shipping_address_id,
            payment_method: payment_method,
            status: :pending,
            order_date: Time.current
          )
          
          # Create order items and calculate totals
          total = 0
          items.each do |item_data|
            product = Product.find(item_data[:product_id])
            
            # Check stock availability
            if product.stock_quantity < item_data[:quantity]
              raise GraphQL::ExecutionError, "Insufficient stock for #{product.name}"
            end
            
            unit_price = product.discount_price || product.price
            item_total = unit_price * item_data[:quantity]
            total += item_total
            
            order.order_items.build(
              product: product,
              quantity: item_data[:quantity],
              unit_price: unit_price,
              total_price: item_total
            )
            
            # Decrease stock
            product.update!(stock_quantity: product.stock_quantity - item_data[:quantity])
          end
          
          # Calculate order totals
          order.subtotal = total
          order.tax_amount = total * 0.08 # 8% tax
          order.shipping_cost = calculate_shipping_cost(total)
          order.total_amount = order.subtotal + order.tax_amount + order.shipping_cost
          
          if order.save
            # Send order confirmation email
            OrderMailer.confirmation_email(order).deliver_later
            
            {
              order: order,
              errors: []
            }
          else
            {
              order: nil,
              errors: order.errors.full_messages
            }
          end
        end
      rescue ActiveRecord::RecordInvalid => e
        {
          order: nil,
          errors: [e.message]
        }
      end
      
      private
      
      def generate_order_number
        "ORD-#{Date.current.strftime('%Y%m%d')}-#{SecureRandom.hex(4).upcase}"
      end
      
      def calculate_shipping_cost(subtotal)
        return 0 if subtotal > 100 # Free shipping over $100
        15 # Standard shipping cost
      end
    end
  end
end

# app/graphql/types/order_item_input_type.rb
module Types
  class OrderItemInputType < Types::BaseInputObject
    argument :product_id, ID, required: true
    argument :quantity, Integer, required: true
  end
end

# app/graphql/mutations/order/update_order_status.rb
module Mutations
  module Order
    class UpdateOrderStatus < BaseMutation
      description "Update order status (admin only)"
      
      argument :id, ID, required: true
      argument :status, String, required: true
      argument :tracking_number, String, required: false
      
      field :order, Types::OrderType, null: true
      field :errors, [String], null: false
      
      def resolve(id:, status:, tracking_number: nil)
        authorize_admin!
        
        order = Order.find(id)
        attributes = { status: status }
        attributes[:tracking_number] = tracking_number if tracking_number
        attributes[:shipped_date] = Time.current if status == 'shipped'
        attributes[:delivered_date] = Time.current if status == 'delivered'
        
        if order.update(attributes)
          # Send status update email
          OrderMailer.status_update_email(order).deliver_later
          
          {
            order: order,
            errors: []
          }
        else
          {
            order: nil,
            errors: order.errors.full_messages
          }
        end
      end
    end
  end
end

# app/graphql/mutations/order/cancel_order.rb
module Mutations
  module Order
    class CancelOrder < BaseMutation
      description "Cancel an order"
      
      argument :id, ID, required: true
      argument :reason, String, required: false
      
      field :order, Types::OrderType, null: true
      field :errors, [String], null: false
      
      def resolve(id:, reason: nil)
        order = Order.find(id)
        authorize_owner!(order)
        
        unless order.can_be_cancelled?
          raise GraphQL::ExecutionError, "Order cannot be cancelled in current status"
        end
        
        ActiveRecord::Base.transaction do
          # Restore stock quantities
          order.order_items.each do |item|
            product = item.product
            product.update!(stock_quantity: product.stock_quantity + item.quantity)
          end
          
          order.update!(
            status: :cancelled,
            cancellation_reason: reason,
            cancelled_at: Time.current
          )
          
          # Send cancellation email
          OrderMailer.cancellation_email(order).deliver_later
        end
        
        {
          order: order,
          errors: []
        }
      rescue ActiveRecord::RecordInvalid => e
        {
          order: nil,
          errors: [e.message]
        }
      end
    end
  end
end
```

### Cart Mutations

```ruby
# app/graphql/mutations/cart/add_to_cart.rb
module Mutations
  module Cart
    class AddToCart < BaseMutation
      description "Add item to cart"
      
      argument :product_id, ID, required: true
      argument :quantity, Integer, required: true, default_value: 1
      
      field :cart_item, Types::CartItemType, null: true
      field :errors, [String], null: false
      
      def resolve(product_id:, quantity:)
        authenticate_user!
        
        product = Product.find(product_id)
        cart = current_user.cart || current_user.create_cart!
        
        # Check if item already in cart
        cart_item = cart.cart_items.find_by(product: product)
        
        if cart_item
          # Update quantity
          new_quantity = cart_item.quantity + quantity
          if new_quantity > product.stock_quantity
            return {
              cart_item: nil,
              errors: ["Not enough stock available"]
            }
          end
          cart_item.update!(quantity: new_quantity)
        else
          # Create new cart item
          if quantity > product.stock_quantity
            return {
              cart_item: nil,
              errors: ["Not enough stock available"]
            }
          end
          
          cart_item = cart.cart_items.create!(
            product: product,
            quantity: quantity,
            unit_price: product.discount_price || product.price
          )
        end
        
        {
          cart_item: cart_item,
          errors: []
        }
      rescue ActiveRecord::RecordInvalid => e
        {
          cart_item: nil,
          errors: [e.message]
        }
      end
    end
  end
end

# app/graphql/mutations/cart/remove_from_cart.rb
module Mutations
  module Cart
    class RemoveFromCart < BaseMutation
      description "Remove item from cart"
      
      argument :cart_item_id, ID, required: true
      
      field :success, Boolean, null: false
      field :errors, [String], null: false
      
      def resolve(cart_item_id:)
        authenticate_user!
        
        cart_item = current_user.cart&.cart_items&.find(cart_item_id)
        
        unless cart_item
          raise GraphQL::ExecutionError, "Cart item not found"
        end
        
        if cart_item.destroy
          {
            success: true,
            errors: []
          }
        else
          {
            success: false,
            errors: cart_item.errors.full_messages
          }
        end
      end
    end
  end
end

# app/graphql/mutations/cart/update_cart_item.rb
module Mutations
  module Cart
    class UpdateCartItem < BaseMutation
      description "Update cart item quantity"
      
      argument :cart_item_id, ID, required: true
      argument :quantity, Integer, required: true
      
      field :cart_item, Types::CartItemType, null: true
      field :errors, [String], null: false
      
      def resolve(cart_item_id:, quantity:)
        authenticate_user!
        
        cart_item = current_user.cart&.cart_items&.find(cart_item_id)
        
        unless cart_item
          raise GraphQL::ExecutionError, "Cart item not found"
        end
        
        if quantity <= 0
          cart_item.destroy
          return {
            cart_item: nil,
            errors: []
          }
        end
        
        if quantity > cart_item.product.stock_quantity
          return {
            cart_item: nil,
            errors: ["Not enough stock available"]
          }
        end
        
        if cart_item.update(quantity: quantity)
          {
            cart_item: cart_item,
            errors: []
          }
        else
          {
            cart_item: nil,
            errors: cart_item.errors.full_messages
          }
        end
      end
    end
  end
end

# app/graphql/types/cart_type.rb
module Types
  class CartType < Types::BaseObject
    field :id, ID, null: false
    field :user, Types::UserType, null: false
    field :cart_items, [Types::CartItemType], null: false
    field :items_count, Integer, null: false
    field :total_amount, Float, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    field :updated_at, GraphQL::Types::ISO8601DateTime, null: false
    
    def items_count
      object.cart_items.sum(:quantity)
    end
    
    def total_amount
      object.cart_items.sum { |item| item.quantity * item.unit_price }
    end
  end
end

# app/graphql/types/cart_item_type.rb
module Types
  class CartItemType < Types::BaseObject
    field :id, ID, null: false
    field :product, Types::ProductType, null: false
    field :quantity, Integer, null: false
    field :unit_price, Float, null: false
    field :total_price, Float, null: false
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false
    
    def total_price
      object.quantity * object.unit_price
    end
  end
end
```

### Mutation Type Root

```ruby
# app/graphql/types/mutation_type.rb
module Types
  class MutationType < Types::BaseObject
    # User mutations
    field :create_user, mutation: Mutations::User::CreateUser
    field :update_user, mutation: Mutations::User::UpdateUser
    field :delete_user, mutation: Mutations::User::DeleteUser
    
    # Product mutations
    field :create_product, mutation: Mutations::Product::CreateProduct
    field :update_product, mutation: Mutations::Product::UpdateProduct
    field :delete_product, mutation: Mutations::Product::DeleteProduct
    
    # Order mutations
    field :create_order, mutation: Mutations::Order::CreateOrder
    field :update_order_status, mutation: Mutations::Order::UpdateOrderStatus
    field :cancel_order, mutation: Mutations::Order::CancelOrder
    
    # Cart mutations
    field :add_to_cart, mutation: Mutations::Cart::AddToCart
    field :remove_from_cart, mutation: Mutations::Cart::RemoveFromCart
    field :update_cart_item, mutation: Mutations::Cart::UpdateCartItem
    field :clear_cart, mutation: Mutations::Cart::ClearCart
    
    # Review mutations
    field :create_review, mutation: Mutations::Review::CreateReview
    field :update_review, mutation: Mutations::Review::UpdateReview
    field :delete_review, mutation: Mutations::Review::DeleteReview
  end
end
```

## Subscriptions

### Subscription Setup

```ruby
# app/graphql/types/subscription_type.rb
module Types
  class SubscriptionType < Types::BaseObject
    field :order_updated, Types::OrderType, null: false do
      description "Subscribe to order status updates"
      argument :order_id, ID, required: true
    end
    
    field :product_stock_updated, Types::ProductType, null: false do
      description "Subscribe to product stock updates"
      argument :product_id, ID, required: true
    end
    
    field :new_message, Types::MessageType, null: false do
      description "Subscribe to new chat messages"
      argument :chat_id, ID, required: true
    end
    
    def order_updated(order_id:)
      # Authorization check
      order = Order.find(order_id)
      unless context[:current_user]&.admin? || order.user == context[:current_user]
        raise GraphQL::ExecutionError, "Access denied"
      end
      
      # Return the order object
      order
    end
    
    def product_stock_updated(product_id:)
      Product.find(product_id)
    end
    
    def new_message(chat_id:)
      # Implementation depends on your chat system
      chat = Chat.find(chat_id)
      
      # Check if user has access to this chat
      unless chat.participants.include?(context[:current_user])
        raise GraphQL::ExecutionError, "Access denied"
      end
      
      # This would be triggered elsewhere in your app
      nil
    end
  end
end

# In your models, trigger subscriptions
class Order < ApplicationRecord
  after_update :trigger_subscription
  
  private
  
  def trigger_subscription
    if saved_change_to_status?
      MyAppSchema.subscriptions.trigger('orderUpdated', { order_id: id }, self)
    end
  end
end

class Product < ApplicationRecord
  after_update :trigger_stock_subscription
  
  private
  
  def trigger_stock_subscription
    if saved_change_to_stock_quantity?
      MyAppSchema.subscriptions.trigger('productStockUpdated', { product_id: id }, self)
    end
  end
end
```

## Authentication and Authorization

### Authentication Setup

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  # Disable CSRF protection for GraphQL API
  skip_before_action :verify_authenticity_token
  
  def execute
    variables = prepare_variables(params[:variables])
    query = params[:query]
    operation_name = params[:operationName]
    
    context = {
      current_user: current_user,
      request: request
    }
    
    result = MyAppSchema.execute(query, 
                                variables: variables, 
                                context: context, 
                                operation_name: operation_name)
    
    render json: result
  rescue => e
    raise e unless Rails.env.development?
    handle_error_in_development(e)
  end
  
  private
  
  def current_user
    # JWT token authentication
    return nil unless request.headers['Authorization']
    
    token = request.headers['Authorization'].split(' ').last
    return nil unless token
    
    begin
      decoded_token = JWT.decode(token, Rails.application.secret_key_base)
      user_id = decoded_token[0]['user_id']
      User.find(user_id)
    rescue JWT::DecodeError, ActiveRecord::RecordNotFound
      nil
    end
  end
  
  def prepare_variables(variables_param)
    case variables_param
    when String
      if variables_param.present?
        JSON.parse(variables_param) || {}
      else
        {}
      end
    when Hash
      variables_param
    when ActionController::Parameters
      variables_param.to_unsafe_hash
    when nil
      {}
    else
      raise ArgumentError, "Unexpected parameter: #{variables_param}"
    end
  end
  
  def handle_error_in_development(e)
    logger.error e.message
    logger.error e.backtrace.join("\n")
    
    render json: { 
      errors: [{ message: e.message, backtrace: e.backtrace }], 
      data: {} 
    }, status: 500
  end
end
```

### Field-Level Authorization

```ruby
# app/graphql/types/base_field.rb
module Types
  class BaseField < GraphQL::Schema::Field
    def resolve_field(obj, args, ctx)
      # Check permissions before resolving
      authorize_field!(obj, ctx)
      super
    end
    
    private
    
    def authorize_field!(obj, ctx)
      # Override in specific fields
      return true unless respond_to?(:authorized?)
      
      unless authorized?(obj, args, ctx)
        raise GraphQL::ExecutionError, "Access denied"
      end
    end
  end
end

# Usage in specific fields
field :admin_notes, String, null: true do
  def authorized?(obj, args, ctx)
    ctx[:current_user]&.admin?
  end
end
```

### Policy-Based Authorization

```ruby
# app/policies/user_policy.rb
class UserPolicy
  attr_reader :current_user, :user
  
  def initialize(current_user, user)
    @current_user = current_user
    @user = user
  end
  
  def show?
    current_user.admin? || current_user == user
  end
  
  def update?
    current_user.admin? || current_user == user
  end
  
  def destroy?
    current_user.admin? || current_user == user
  end
end

# Usage in resolvers
def resolve(id:)
  user = User.find(id)
  
  unless UserPolicy.new(context[:current_user], user).show?
    raise GraphQL::ExecutionError, "Access denied"
  end
  
  user
end
```

## Error Handling

### Custom Error Classes

```ruby
# app/graphql/errors/authentication_error.rb
module Errors
  class AuthenticationError < GraphQL::ExecutionError
    def initialize(message = "Authentication required")
      super(message)
    end
    
    def to_h
      super.merge("extensions" => { "code" => "AUTHENTICATION_ERROR" })
    end
  end
end

# app/graphql/errors/authorization_error.rb
module Errors
  class AuthorizationError < GraphQL::ExecutionError
    def initialize(message = "Access denied")
      super(message)
    end
    
    def to_h
      super.merge("extensions" => { "code" => "AUTHORIZATION_ERROR" })
    end
  end
end

# app/graphql/errors/validation_error.rb
module Errors
  class ValidationError < GraphQL::ExecutionError
    def initialize(model)
      @model = model
      super("Validation failed: #{model.errors.full_messages.join(', ')}")
    end
    
    def to_h
      super.merge(
        "extensions" => {
          "code" => "VALIDATION_ERROR",
          "details" => @model.errors.messages
        }
      )
    end
  end
end
```

### Schema Error Handling

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  # ... other configuration
  
  rescue_from(ActiveRecord::RecordNotFound) do |error|
    GraphQL::ExecutionError.new("Record not found")
  end
  
  rescue_from(ActiveRecord::RecordInvalid) do |error|
    Errors::ValidationError.new(error.record)
  end
  
  rescue_from(StandardError) do |error|
    if Rails.env.production?
      # Log the error but don't expose details
      Rails.logger.error "GraphQL Error: #{error.message}"
      Rails.logger.error error.backtrace.join("\n")
      GraphQL::ExecutionError.new("An unexpected error occurred")
    else
      # In development, show the full error
      GraphQL::ExecutionError.new("#{error.class}: #{error.message}")
    end
  end
end
```

## Performance Optimization

### N+1 Query Prevention with Batch Loading

```ruby
# Install graphql-batch gem
gem 'graphql-batch'

# app/graphql/loaders/record_loader.rb
class RecordLoader < GraphQL::Batch::Loader
  def initialize(model)
    @model = model
  end
  
  def perform(ids)
    records = @model.where(id: ids).index_by(&:id)
    ids.each { |id| fulfill(id, records[id]) }
  end
end

# app/graphql/loaders/association_loader.rb
class AssociationLoader < GraphQL::Batch::Loader
  def initialize(model, association_name)
    @model = model
    @association_name = association_name
  end
  
  def perform(ids)
    records = @model.includes(@association_name).where(id: ids).index_by(&:id)
    ids.each { |id| fulfill(id, records[id]&.send(@association_name)) }
  end
end

# Usage in types
def reviews
  AssociationLoader.for(Product, :reviews).load(object.id)
end

def user
  RecordLoader.for(User).load(object.user_id)
end
```

### Database Query Optimization

```ruby
# app/graphql/types/user_type.rb
module Types
  class UserType < Types::BaseObject
    # Use includes to preload associations
    field :orders, [Types::OrderType], null: false, preload: :orders
    
    # Batch complex calculations
    field :total_spent, Float, null: false
    
    def total_spent
      # Use BatchLoader to avoid N+1 queries
      BatchLoader::GraphQL.for(object.id).batch do |user_ids, loader|
        totals = Order.where(user_id: user_ids, status: :delivered)
                     .group(:user_id)
                     .sum(:total_amount)
        user_ids.each { |id| loader.call(id, totals[id] || 0.0) }
      end
    end
  end
end
```

### Query Complexity Analysis

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  # Analyze query complexity
  query_analyzer GraphQL::Analysis::QueryComplexity.new(max_complexity: 200)
  
  # Analyze query depth
  query_analyzer GraphQL::Analysis::QueryDepth.new(max_depth: 15)
  
  # Add field complexity
  field :expensive_calculation, String, null: false, complexity: 10
end
```

### Caching

```ruby
# app/graphql/types/product_type.rb
module Types
  class ProductType < Types::BaseObject
    field :average_rating, Float, null: true
    
    def average_rating
      # Cache expensive calculations
      Rails.cache.fetch("product_#{object.id}_average_rating", expires_in: 1.hour) do
        object.reviews.average(:rating)&.round(2)
      end
    end
  end
end
```

## Testing

### RSpec Setup

```ruby
# spec/rails_helper.rb
RSpec.configure do |config|
  config.include GraphqlHelper, type: :graphql
end

# spec/support/graphql_helper.rb
module GraphqlHelper
  def execute_graphql(query, variables: {}, context: {})
    MyAppSchema.execute(query, variables: variables, context: context)
  end
  
  def graphql_data
    @result['data']
  end
  
  def graphql_errors
    @result['errors']
  end
  
  def expect_graphql_error(error_message)
    expect(graphql_errors).to include(
      hash_including('message' => error_message)
    )
  end
end
```

### Query Testing

```ruby
# spec/graphql/queries/user_query_spec.rb
RSpec.describe 'User Query', type: :graphql do
  let(:user) { create(:user) }
  let(:admin) { create(:user, role: :admin) }
  
  describe 'currentUser' do
    let(:query) do
      <<~GQL
        query {
          currentUser {
            id
            email
            fullName
            role
          }
        }
      GQL
    end
    
    context 'when authenticated' do
      it 'returns current user data' do
        @result = execute_graphql(query, context: { current_user: user })
        
        expect(graphql_data['currentUser']).to include(
          'id' => user.id.to_s,
          'email' => user.email,
          'fullName' => "#{user.first_name} #{user.last_name}",
          'role' => user.role
        )
      end
    end
    
    context 'when not authenticated' do
      it 'returns null' do
        @result = execute_graphql(query, context: {})
        
        expect(graphql_data['currentUser']).to be_nil
      end
    end
  end
  
  describe 'user(id:)' do
    let(:query) do
      <<~GQL
        query($id: ID!) {
          user(id: $id) {
            id
            email
            fullName
          }
        }
      GQL
    end
    
    context 'when admin' do
      it 'returns user data' do
        @result = execute_graphql(
          query, 
          variables: { id: user.id }, 
          context: { current_user: admin }
        )
        
        expect(graphql_data['user']['id']).to eq(user.id.to_s)
      end
    end
    
    context 'when not admin' do
      it 'returns access denied error' do
        @result = execute_graphql(
          query, 
          variables: { id: user.id }, 
          context: { current_user: user }
        )
        
        expect_graphql_error('Admin access required')
      end
    end
  end
end
```

### Mutation Testing

```ruby
# spec/graphql/mutations/create_user_spec.rb
RSpec.describe Mutations::User::CreateUser, type: :graphql do
  let(:mutation) do
    <<~GQL
      mutation($input: CreateUserInput!) {
        createUser(input: $input) {
          user {
            id
            email
            firstName
            lastName
          }
          errors
        }
      }
    GQL
  end
  
  let(:valid_attributes) do
    {
      email: 'test@example.com',
      password: 'Password123!',
      firstName: 'John',
      lastName: 'Doe'
    }
  end
  
  context 'with valid attributes' do
    it 'creates a user' do
      expect {
        @result = execute_graphql(mutation, variables: { input: valid_attributes })
      }.to change(User, :count).by(1)
      
      user_data = graphql_data['createUser']['user']
      expect(user_data['email']).to eq('test@example.com')
      expect(user_data['firstName']).to eq('John')
      expect(user_data['lastName']).to eq('Doe')
      expect(graphql_data['createUser']['errors']).to be_empty
    end
  end
  
  context 'with invalid attributes' do
    let(:invalid_attributes) { valid_attributes.merge(email: 'invalid') }
    
    it 'returns validation errors' do
      expect {
        @result = execute_graphql(mutation, variables: { input: invalid_attributes })
      }.not_to change(User, :count)
      
      expect(graphql_data['createUser']['user']).to be_nil
      expect(graphql_data['createUser']['errors']).to include('Email is invalid')
    end
  end
end
```

### Integration Testing

```ruby
# spec/requests/graphql_spec.rb
RSpec.describe 'GraphQL API', type: :request do
  let(:user) { create(:user) }
  let(:token) { generate_jwt_token(user) }
  
  describe 'POST /graphql' do
    let(:query) do
      <<~GQL
        query {
          currentUser {
            id
            email
          }
        }
      GQL
    end
    
    context 'with valid token' do
      it 'returns user data' do
        post '/graphql', 
             params: { query: query },
             headers: { 'Authorization' => "Bearer #{token}" }
        
        expect(response).to have_http_status(:ok)
        json = JSON.parse(response.body)
        expect(json['data']['currentUser']['id']).to eq(user.id.to_s)
      end
    end
    
    context 'without token' do
      it 'returns null for current user' do
        post '/graphql', params: { query: query }
        
        expect(response).to have_http_status(:ok)
        json = JSON.parse(response.body)
        expect(json['data']['currentUser']).to be_nil
      end
    end
  end
end

def generate_jwt_token(user)
  JWT.encode({ user_id: user.id }, Rails.application.secret_key_base)
end
```

## Production Considerations

### Security

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  # Query timeout
  query_analyzer GraphQL::Analysis::QueryComplexity.new(max_complexity: 200)
  query_analyzer GraphQL::Analysis::QueryDepth.new(max_depth: 15)
  
  # Disable introspection in production
  disable_introspection_entry_points if Rails.env.production?
  
  # Rate limiting
  use GraphQL::Pro::RateLimiter, redis: Redis.current
  
  # Query logging
  query_analyzer GraphQL::Analysis::Logging.new
end
```

### Rate Limiting

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  before_action :rate_limit_graphql
  
  private
  
  def rate_limit_graphql
    # Simple rate limiting based on IP
    key = "graphql_rate_limit:#{request.remote_ip}"
    current_requests = Rails.cache.read(key) || 0
    
    if current_requests >= 100 # 100 requests per minute
      render json: { 
        errors: [{ message: "Rate limit exceeded" }] 
      }, status: 429
      return
    end
    
    Rails.cache.write(key, current_requests + 1, expires_in: 1.minute)
  end
end
```

### Monitoring and Logging

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  # Custom tracer for monitoring
  tracer GraphQL::Tracing::PrometheusTracer.new
  
  # Query logging
  query_analyzer GraphQL::Analysis::AST::QueryComplexity.new(max_complexity: 200)
  
  # Custom instrumentation
  instrument :field, GraphQL::Tracing::ScoutTracer.new
end

# app/graphql/tracers/custom_tracer.rb
class CustomTracer
  def trace(key, data)
    start_time = Time.current
    result = yield
    duration = Time.current - start_time
    
    # Log slow queries
    if duration > 1.second && key == "execute_query"
      Rails.logger.warn "Slow GraphQL Query: #{duration}s - #{data[:query].query_string}"
    end
    
    # Send metrics to monitoring service
    StatsD.histogram('graphql.query.duration', duration, tags: ["operation:#{key}"])
    
    result
  end
end
```

### Deployment Configuration

```ruby
# config/application.rb
module MyApp
  class Application < Rails::Application
    # GraphQL configuration
    config.graphql = config_for(:graphql)
  end
end

# config/graphql.yml
development:
  introspection_enabled: true
  query_depth_limit: 20
  query_complexity_limit: 300
  rate_limit_per_minute: 1000

test:
  introspection_enabled: true
  query_depth_limit: 20
  query_complexity_limit: 300
  rate_limit_per_minute: 10000

production:
  introspection_enabled: false
  query_depth_limit: 15
  query_complexity_limit: 200
  rate_limit_per_minute: 100
```

### Docker Configuration

```dockerfile
# Dockerfile
FROM ruby:3.1

WORKDIR /app

# Install dependencies
COPY Gemfile Gemfile.lock ./
RUN bundle install

# Copy application
COPY . .

# Precompile assets (if using Rails assets)
RUN RAILS_ENV=production rails assets:precompile

# Expose port
EXPOSE 3000

# Start server
CMD ["rails", "server", "-b", "0.0.0.0"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp_production
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - db
      - redis
    volumes:
      - ./storage:/app/storage

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=myapp_production
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## Advanced Patterns

### Federation (for Microservices)

```ruby
# Gemfile
gem 'apollo-federation'

# app/graphql/my_app_schema.rb
class MyAppSchema < ApolloFederation::Schema
  # Include federation directives
  include ApolloFederation::Schema
  
  # Define federated types
  orphan_types Types::UserType, Types::ProductType
end

# app/graphql/types/user_type.rb
module Types
  class UserType < Types::BaseObject
    # Mark as federated entity
    extend_type
    
    key fields: "id"
    
    field :id, ID, null: false, external: true
    field :email, String, null: false
    field :orders, [Types::OrderType], null: false
    
    # Resolver for federation
    def self.resolve_reference(object, context)
      User.find(object.id)
    end
  end
end
```

### Real-time Features with ActionCable

```ruby
# app/channels/graphql_channel.rb
class GraphqlChannel < ApplicationCable::Channel
  def subscribed
    @subscription_ids = []
  end
  
  def execute(data)
    result = MyAppSchema.execute(
      query: data['query'],
      variables: data['variables'],
      context: {
        current_user: current_user,
        channel: self
      },
      operation_name: data['operationName']
    )
    
    if result.subscription?
      @subscription_ids << result.context[:subscription_id]
    else
      transmit(result.to_h)
    end
  end
  
  def unsubscribed
    @subscription_ids.each do |sid|
      MyAppSchema.subscriptions.delete_subscription(sid)
    end
  end
end

# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  use GraphQL::Subscriptions::ActionCableSubscriptions
  
  subscription(Types::SubscriptionType)
end
```

### File Upload Support

```ruby
# Gemfile
gem 'graphql-upload'

# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  # Enable file uploads
  scalar_class GraphqlUpload::Upload
end

# app/graphql/mutations/product/upload_images.rb
module Mutations
  module Product
    class UploadImages < BaseMutation
      argument :product_id, ID, required: true
      argument :images, [GraphqlUpload::Upload], required: true
      
      field :product, Types::ProductType, null: true
      field :errors, [String], null: false
      
      def resolve(product_id:, images:)
        product = Product.find(product_id)
        
        # Check permissions
        unless current_user&.admin? || product.vendor == current_user
          raise GraphQL::ExecutionError, "Access denied"
        end
        
        images.each do |image|
          # Validate file type
          unless image.content_type.start_with?('image/')
            return {
              product: nil,
              errors: ["Invalid file type: #{image.filename}"]
            }
          end
          
          # Attach image
          product.images.attach(image)
        end
        
        {
          product: product,
          errors: []
        }
      end
    end
  end
end
```

### Custom Scalar Types

```ruby
# app/graphql/types/date_time_type.rb
module Types
  class DateTimeType < GraphQL::Schema::Scalar
    description "ISO 8601 DateTime"
    
    def self.coerce_input(input_value, context)
      Time.zone.parse(input_value)
    rescue ArgumentError
      raise GraphQL::CoercionError, "#{input_value.inspect} is not a valid DateTime"
    end
    
    def self.coerce_result(ruby_value, context)
      ruby_value.iso8601
    end
  end
end

# app/graphql/types/money_type.rb
module Types
  class MoneyType < GraphQL::Schema::Scalar
    description "Money amount in cents"
    
    def self.coerce_input(input_value, context)
      case input_value
      when String
        (input_value.to_f * 100).to_i
      when Numeric
        (input_value * 100).to_i
      else
        raise GraphQL::CoercionError, "#{input_value.inspect} is not a valid Money amount"
      end
    end
    
    def self.coerce_result(ruby_value, context)
      (ruby_value / 100.0).round(2)
    end
  end
end
```

### GraphQL Playground Setup

```ruby
# config/routes.rb
Rails.application.routes.draw do
  if Rails.env.development?
    mount GraphiQL::Rails::Engine, at: "/graphiql", graphql_path: "/graphql"
  end
  
  post "/graphql", to: "graphql#execute"
end

# app/views/graphiql/rails/editors/show.html.erb
<%
  headers = {}
  if current_user
    token = JWT.encode({ user_id: current_user.id }, Rails.application.secret_key_base)
    headers['Authorization'] = "Bearer #{token}"
  end
%>

<%= content_for :graphiql_headers, headers.to_json %>
```

## Best Practices Summary

### Schema Design
- Use descriptive field and type names
- Implement proper error handling
- Design for client needs, not database structure
- Use connections for pagination
- Implement proper authorization at field level

### Performance
- Use batch loading to prevent N+1 queries
- Implement query complexity analysis
- Cache expensive computations
- Use database indexes appropriately
- Monitor query performance

### Security
- Implement authentication and authorization
- Disable introspection in production
- Use rate limiting
- Validate all inputs
- Implement proper CORS policies

### Testing
- Test queries, mutations, and subscriptions
- Test error scenarios
- Use integration tests for complete workflows
- Mock external services
- Test authorization scenarios

### Production
- Monitor query performance
- Log slow queries
- Implement proper error tracking
- Use CDN for static assets
- Implement proper caching strategies

This comprehensive guide covers all aspects of implementing GraphQL in Ruby on Rails applications, from basic setup to advanced production patterns. Each section includes practical examples that can be adapted to your specific use case.