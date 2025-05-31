# Rails Database Query Exercises - Generic Examples

## 📚 **E-commerce Store Models**

```ruby
class User < ApplicationRecord
  has_many :orders
  has_many :reviews
  has_one :profile
  has_many :cart_items
end

class Product < ApplicationRecord
  belongs_to :category
  has_many :order_items
  has_many :reviews
  has_many :orders, through: :order_items
end

class Category < ApplicationRecord
  has_many :products
end

class Order < ApplicationRecord
  belongs_to :user
  has_many :order_items
  has_many :products, through: :order_items
  
  enum status: { pending: 0, shipped: 1, delivered: 2, cancelled: 3 }
end

class OrderItem < ApplicationRecord
  belongs_to :order
  belongs_to :product
end

class Review < ApplicationRecord
  belongs_to :user
  belongs_to :product
end
```

---

## 🟢 **BEGINNER EXERCISES**

### Exercise 1: Basic Where Clauses
**Find all products that cost more than $50**

```ruby
# Your answer:
Product.where("price > ?", 50)
# or
Product.where("price > 50")
```

### Exercise 2: Simple Associations
**Get all orders for a user with ID 5**

```ruby
# Your answer:
Order.where(user_id: 5)
# or
User.find(5).orders
```

### Exercise 3: Count Records
**How many products are in the "Electronics" category?**

```ruby
# Your answer:
Product.joins(:category).where(categories: { name: "Electronics" }).count
```

### Exercise 4: Date Filtering
**Find all orders placed in the last 30 days**

```ruby
# Your answer:
Order.where(created_at: 30.days.ago..Time.current)
# or
Order.where("created_at > ?", 30.days.ago)
```

### Exercise 5: Enum Filtering
**Find all delivered orders**

```ruby
# Your answer:
Order.delivered
# or
Order.where(status: :delivered)
# or
Order.where(status: 2)
```

---

## 🟡 **INTERMEDIATE EXERCISES**

### Exercise 6: Multiple Joins
**Find all users who have ordered products from the "Books" category**

```ruby
# Your answer:
User.joins(orders: { order_items: { product: :category } })
    .where(categories: { name: "Books" })
    .distinct
```

### Exercise 7: Aggregation
**What's the total revenue from all delivered orders?**

```ruby
# Your answer:
OrderItem.joins(:order)
         .where(orders: { status: :delivered })
         .sum("quantity * price")

# Alternative:
Order.delivered
     .joins(:order_items)
     .sum("order_items.quantity * order_items.price")
```

### Exercise 8: Group By with Aggregation
**Total sales amount per category**

```ruby
# Your answer:
Category.joins(products: { order_items: :order })
        .where(orders: { status: :delivered })
        .group("categories.name")
        .sum("order_items.quantity * order_items.price")
```

### Exercise 9: Having Clause
**Find categories with more than 10 products**

```ruby
# Your answer:
Category.joins(:products)
        .group("categories.id", "categories.name")
        .having("COUNT(products.id) > 10")
```

### Exercise 10: Complex Date Queries
**Find users who placed orders in both January AND February 2024**

```ruby
# Your answer:
jan_users = User.joins(:orders)
                .where(orders: { created_at: Date.new(2024, 1, 1)..Date.new(2024, 1, 31) })
                .pluck(:id)

feb_users = User.joins(:orders)
                .where(orders: { created_at: Date.new(2024, 2, 1)..Date.new(2024, 2, 29) })
                .pluck(:id)

User.where(id: jan_users & feb_users)

# Alternative with subquery:
User.where(
  id: User.joins(:orders)
          .where(orders: { created_at: Date.new(2024, 1, 1)..Date.new(2024, 1, 31) })
          .select(:id)
).where(
  id: User.joins(:orders)
          .where(orders: { created_at: Date.new(2024, 2, 1)..Date.new(2024, 2, 29) })
          .select(:id)
)
```

---

## 📝 **BLOG/CMS MODELS**

```ruby
class Author < ApplicationRecord
  has_many :posts
  has_many :comments, through: :posts
end

class Post < ApplicationRecord
  belongs_to :author
  belongs_to :category
  has_many :comments
  has_many :post_tags
  has_many :tags, through: :post_tags
  
  enum status: { draft: 0, published: 1, archived: 2 }
end

class Comment < ApplicationRecord
  belongs_to :post
  belongs_to :user
end

class Tag < ApplicationRecord
  has_many :post_tags
  has_many :posts, through: :post_tags
end
```

### Exercise 11: Many-to-Many Relationships
**Find all posts tagged with "Ruby" or "Rails"**

```ruby
# Your answer:
Post.joins(:tags)
    .where(tags: { name: ["Ruby", "Rails"] })
    .distinct
```

### Exercise 12: Nested Associations
**Find all authors who have posts with more than 5 comments**

```ruby
# Your answer:
Author.joins(posts: :comments)
      .group("authors.id")
      .having("COUNT(comments.id) > 5")
      .distinct
```

### Exercise 13: Complex Filtering
**Find published posts from the last week that have at least 3 comments**

```ruby
# Your answer:
Post.published
    .where(created_at: 1.week.ago..Time.current)
    .joins(:comments)
    .group("posts.id")
    .having("COUNT(comments.id) >= 3")
```

---

## 🔴 **ADVANCED EXERCISES**

### Exercise 14: Subqueries
**Find products that have never been ordered**

```ruby
# Your answer:
Product.where.not(
  id: OrderItem.select(:product_id).distinct
)

# Alternative:
Product.left_joins(:order_items)
       .where(order_items: { id: nil })
```

### Exercise 15: Window Functions
**Rank products by total sales within each category**

```ruby
# Your answer:
Product.joins(:category, :order_items)
       .joins("JOIN orders ON orders.id = order_items.order_id")
       .where(orders: { status: :delivered })
       .group("products.id", "categories.name", "products.name")
       .select("
         products.*,
         categories.name as category_name,
         SUM(order_items.quantity * order_items.price) as total_sales,
         RANK() OVER (
           PARTITION BY categories.id 
           ORDER BY SUM(order_items.quantity * order_items.price) DESC
         ) as sales_rank
       ")
```

### Exercise 16: Complex Aggregation with Multiple Conditions
**Monthly sales report: total orders, total revenue, average order value**

```ruby
# Your answer:
Order.delivered
     .joins(:order_items)
     .group("DATE_TRUNC('month', orders.created_at)")
     .select("
       DATE_TRUNC('month', orders.created_at) as month,
       COUNT(DISTINCT orders.id) as total_orders,
       SUM(order_items.quantity * order_items.price) as total_revenue,
       AVG(order_items.quantity * order_items.price) as avg_order_value
     ")
```

---

## 🏪 **RESTAURANT/FOOD DELIVERY MODELS**

```ruby
class Restaurant < ApplicationRecord
  has_many :menu_items
  has_many :orders
  belongs_to :cuisine_type
end

class MenuItem < ApplicationRecord
  belongs_to :restaurant
  has_many :order_items
end

class Customer < ApplicationRecord
  has_many :orders
  has_many :reviews
end

class Order < ApplicationRecord
  belongs_to :customer
  belongs_to :restaurant
  has_many :order_items
  
  enum status: { placed: 0, preparing: 1, out_for_delivery: 2, delivered: 3, cancelled: 4 }
end
```

### Exercise 17: Restaurant Analytics
**Find restaurants with average rating above 4.0 and more than 100 orders**

```ruby
# Your answer:
Restaurant.joins(:orders, :reviews)
          .group("restaurants.id")
          .having("AVG(reviews.rating) > 4.0")
          .having("COUNT(DISTINCT orders.id) > 100")
```

### Exercise 18: Customer Behavior
**Find customers who have ordered from the same restaurant more than 5 times**

```ruby
# Your answer:
Customer.joins(:orders)
        .group("customers.id", "orders.restaurant_id")
        .having("COUNT(orders.id) > 5")
        .select("customers.*, orders.restaurant_id, COUNT(orders.id) as order_count")
```

---

## 🎓 **UNIVERSITY MODELS**

```ruby
class Student < ApplicationRecord
  has_many :enrollments
  has_many :courses, through: :enrollments
  belongs_to :major
end

class Course < ApplicationRecord
  belongs_to :department
  has_many :enrollments
  has_many :students, through: :enrollments
end

class Enrollment < ApplicationRecord
  belongs_to :student
  belongs_to :course
end

class Department < ApplicationRecord
  has_many :courses
  belongs_to :faculty
end
```

### Exercise 19: Academic Analytics
**Find departments with the highest average course enrollment**

```ruby
# Your answer:
Department.joins(courses: :enrollments)
          .group("departments.id", "departments.name")
          .select("
            departments.*,
            AVG(enrollment_count.count) as avg_enrollment
          ")
          .joins("
            JOIN (
              SELECT courses.id, COUNT(enrollments.id) as count
              FROM courses 
              LEFT JOIN enrollments ON enrollments.course_id = courses.id
              GROUP BY courses.id
            ) enrollment_count ON enrollment_count.id = courses.id
          ")
          .order("avg_enrollment DESC")

# Simpler version:
Department.joins(courses: :enrollments)
          .group("departments.id", "departments.name")
          .select("departments.*, COUNT(enrollments.id)::float / COUNT(DISTINCT courses.id) as avg_enrollment")
          .order("avg_enrollment DESC")
```

### Exercise 20: Student Performance
**Find students enrolled in more than 6 courses this semester**

```ruby
# Your answer:
Student.joins(:enrollments)
       .where(enrollments: { semester: "Fall 2024" })
       .group("students.id")
       .having("COUNT(enrollments.id) > 6")
```

---

## 🎯 **PRACTICE WORKFLOW**

### 1. **Start Simple, Build Complex**
```ruby
# Always start with the base model:
Product.all

# Add one filter at a time:
Product.where("price > 50")
Product.where("price > 50").joins(:category)
Product.where("price > 50").joins(:category).where(categories: { name: "Electronics" })
```

### 2. **Test in Console**
```ruby
rails console

# Check your SQL:
query = Product.joins(:category).where(categories: { name: "Electronics" })
puts query.to_sql
puts query.count  # Make sure it works
```

### 3. **Understand Performance**
```ruby
# Use EXPLAIN to see query performance:
Product.joins(:category).where(categories: { name: "Electronics" }).explain
```

### 4. **Practice Different Approaches**
```ruby
# Method 1: Joins
Product.joins(:category).where(categories: { name: "Electronics" })

# Method 2: Subquery
Product.where(category_id: Category.where(name: "Electronics").select(:id))

# Method 3: Through association
electronics_category = Category.find_by(name: "Electronics")
electronics_category.products
```

---

## 📈 **Key Patterns to Master**

1. **Basic Filtering**: `where`, `where.not`
2. **Associations**: `joins`, `left_joins`, `includes`
3. **Aggregations**: `count`, `sum`, `average`, `group`, `having`
4. **Date Ranges**: `where(created_at: range)`
5. **Subqueries**: `where(id: Model.select(:id))`
6. **Performance**: `includes` vs `joins`, `select` for limiting columns

**Pro Tip**: Master these 20 exercises and you'll be able to handle 90% of real-world Rails queries! 🚀