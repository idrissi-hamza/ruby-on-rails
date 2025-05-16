# Rails API Models and Database Architecture

## Model Associations

In Rails, model associations define the relationships between different database tables. Here's a detailed breakdown of the associations in our Task Manager API:

### User Model

```ruby
class User < ApplicationRecord
  has_secure_password
  
  has_many :projects, dependent: :destroy
  has_many :comments, dependent: :destroy
  
  # Add all tasks for a user through their projects
  has_many :tasks, through: :projects
  
  validates :name, presence: true
  validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
end
```

**Explanation:**

1. **`has_secure_password`**: This Rails method provides password hashing using bcrypt. It:
   - Adds virtual attributes (`password` and `password_confirmation`)
   - Validates that password is present on creation
   - Validates that password_confirmation matches password (if provided)
   - Adds an `authenticate` method that returns the user if password is correct

2. **`has_many :projects, dependent: :destroy`**:
   - Creates a one-to-many relationship where a user can have multiple projects
   - The `dependent: :destroy` option ensures that when a user is deleted, all their projects are automatically deleted (cascading delete)
   - Provides helper methods like `user.projects`, `user.projects.create(attributes)`, etc.

3. **`has_many :comments, dependent: :destroy`**:
   - Similar to projects, establishes that a user can have many comments
   - Comments will be deleted when the user is deleted

4. **`has_many :tasks, through: :projects`**:
   - This is a "has-many-through" association
   - It creates a logical relationship between users and tasks via the projects table
   - Allows direct access to all tasks belonging to all projects owned by the user
   - Example: `user.tasks` will return all tasks across all of the user's projects

5. **Validations**:
   - `validates :name, presence: true`: Ensures name cannot be blank
   - `validates :email, presence: true, uniqueness: true`: Ensures email is required and unique
   - The `format: { with: URI::MailTo::EMAIL_REGEXP }` option validates that the email follows a standard email format

### Project Model

```ruby
class Project < ApplicationRecord
  belongs_to :user
  has_many :tasks, dependent: :destroy
  
  validates :title, presence: true
  validates :status, inclusion: { in: ['planning', 'in_progress', 'completed'] }
end
```

**Explanation:**

1. **`belongs_to :user`**:
   - Establishes the inverse of the has_many relationship
   - A project must belong to exactly one user
   - Creates methods like `project.user`, `project.user=`, `project.build_user`, etc.
   - Since Rails 5, this creates an implicit validation that `user_id` must be present

2. **`has_many :tasks, dependent: :destroy`**:
   - Creates a one-to-many relationship with tasks
   - When a project is deleted, all associated tasks are automatically deleted

3. **Validations**:
   - `validates :title, presence: true`: Ensures title is required
   - `validates :status, inclusion: { in: ['planning', 'in_progress', 'completed'] }`: 
     - Ensures that status is one of the three allowed values
     - This creates a domain constraint at the application level

### Task Model

```ruby
class Task < ApplicationRecord
  belongs_to :project
  has_many :comments, dependent: :destroy
  
  validates :title, presence: true
  validates :status, inclusion: { in: ['todo', 'in_progress', 'done'] }
  validates :priority, inclusion: { in: ['low', 'medium', 'high'] }
  
  # Delegate user to project
  delegate :user, to: :project
end
```

**Explanation:**

1. **`belongs_to :project`**:
   - Each task belongs to exactly one project
   - Creates an implicit validation requiring a valid project_id

2. **`has_many :comments, dependent: :destroy`**:
   - A task can have multiple comments
   - Comments are deleted when their parent task is deleted

3. **`delegate :user, to: :project`**:
   - This is a Ruby delegation pattern that creates a `user` method on tasks
   - Instead of having to call `task.project.user`, you can simply call `task.user`
   - This simplifies code and maintains proper object relationships
   - It doesn't create a direct database association, just a method shortcut

4. **Validations**:
   - Title must be present
   - Status must be one of: 'todo', 'in_progress', 'done'
   - Priority must be one of: 'low', 'medium', 'high'

### Comment Model

```ruby
class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :task
  
  validates :content, presence: true
end
```

**Explanation:**

1. **`belongs_to :user`**:
   - Each comment is authored by one user
   - Creates methods to access the associated user

2. **`belongs_to :task`**:
   - Each comment belongs to one task
   - Creates methods to access the associated task

3. **`validates :content, presence: true`**:
   - Ensures that a comment must have content

## Database Migrations

Migrations define the structure of your database tables. Let's look at the enhanced migrations:

### Users Migration

```ruby
class CreateUsers < ActiveRecord::Migration[7.0]
  def change
    create_table :users do |t|
      t.string :name, null: false
      t.string :email, null: false
      t.string :password_digest, null: false

      t.timestamps
    end
    add_index :users, :email, unique: true
  end
end
```

**Explanation:**

1. **`create_table :users`**:
   - Creates a new table named "users"

2. **Column definitions**:
   - `t.string :name, null: false`: String column that cannot be NULL
   - `t.string :email, null: false`: String column that cannot be NULL
   - `t.string :password_digest, null: false`: For storing bcrypt hashed passwords

3. **`t.timestamps`**:
   - Adds `created_at` and `updated_at` datetime columns
   - Rails automatically manages these timestamps

4. **`add_index :users, :email, unique: true`**:
   - Creates a database-level unique index on the email column
   - Improves query performance when searching by email
   - Enforces uniqueness at the database level (additional to model validation)
   - Prevents race conditions that could allow duplicate emails

### Projects Migration

```ruby
class CreateProjects < ActiveRecord::Migration[7.0]
  def change
    create_table :projects do |t|
      t.string :title, null: false
      t.text :description
      t.string :status, default: 'planning'
      t.references :user, null: false, foreign_key: true

      t.timestamps
    end
    add_index :projects, [:user_id, :title]
  end
end
```

**Explanation:**

1. **Column definitions**:
   - `t.string :title, null: false`: String column that cannot be NULL
   - `t.text :description`: Text column for longer content, can be NULL
   - `t.string :status, default: 'planning'`: String with a default value

2. **`t.references :user, null: false, foreign_key: true`**:
   - Creates a `user_id` column that references the users table
   - `null: false` ensures the column cannot be NULL
   - `foreign_key: true` adds a foreign key constraint at the database level
   - This constraint ensures referential integrity (can't reference non-existent users)

3. **`add_index :projects, [:user_id, :title]`**:
   - Creates a composite index on both `user_id` and `title`
   - Improves performance when:
     - Querying for a user's projects
     - Searching for a project by title within a user's projects
   - Useful for queries like `Project.where(user_id: 5, title: 'My Project')`

### Tasks Migration

```ruby
class CreateTasks < ActiveRecord::Migration[7.0]
  def change
    create_table :tasks do |t|
      t.string :title, null: false
      t.text :description
      t.datetime :due_date
      t.string :status, default: 'todo'
      t.string :priority, default: 'medium'
      t.references :project, null: false, foreign_key: true

      t.timestamps
    end
    add_index :tasks, [:project_id, :status]
  end
end
```

**Explanation:**

1. **Column definitions**:
   - `t.string :title, null: false`: Required string column
   - `t.text :description`: Optional text column
   - `t.datetime :due_date`: DateTime column for task deadlines
   - `t.string :status, default: 'todo'`: String with default value
   - `t.string :priority, default: 'medium'`: String with default value

2. **`t.references :project, null: false, foreign_key: true`**:
   - Creates a `project_id` column that references the projects table
   - Adds a foreign key constraint to ensure database integrity

3. **`add_index :tasks, [:project_id, :status]`**:
   - Creates a composite index on `project_id` and `status`
   - Optimizes queries like "show me all 'in_progress' tasks for Project X"
   - This index is particularly useful for filtered views of tasks

## Advanced Database Design Concepts

### 1. **Database Constraints vs. Application Validations**

- **Database constraints** (NULL constraints, foreign keys, unique indexes) operate at the database level and are the last line of defense
- **Application validations** operate in your Rails code and provide better user feedback
- For critical data integrity, use both approaches

### 2. **Indexing Strategy**

The migrations use three types of indexes:
- **Single-column index**: `add_index :users, :email, unique: true`
- **Composite indexes**: `add_index :projects, [:user_id, :title]`
- **Foreign key indexes**: Automatically created by `t.references`

Indexes speed up queries but add overhead on writes, so they're strategically placed on:
- Columns frequently used in WHERE clauses
- Foreign keys (user_id, project_id)
- Columns used for sorting (ORDER BY)

### 3. **Soft Deletes (Alternative Approach)**

Instead of using `dependent: :destroy` for hard deletes, you could implement soft deletes:

```ruby
# Add deleted_at column to tables
add_column :projects, :deleted_at, :datetime, null: true
add_index :projects, :deleted_at

# In the model
class Project < ApplicationRecord
  # Default scope excludes deleted records
  default_scope { where(deleted_at: nil) }
  
  # Custom method for soft delete
  def soft_delete
    update(deleted_at: Time.current)
  end
end
```

### 4. **The Role of Migrations**

Migrations serve as:
- **Version control** for your database schema
- A way to **share schema changes** with your team
- A method to **ensure consistency** across environments

That's why they include:
- Column constraints (`null: false`)
- Default values (`default: 'todo'`)
- Indexes for performance
- Foreign keys for integrity

By running `rails db:migrate`, all these structure definitions are applied to your database, creating a robust foundation for your API.

## Best Practices for Rails Model Development

### 1. **Keep Models Thin, Controllers Thinner**

Business logic should go in:
- Service objects for complex operations
- Model concerns for shared functionality
- Custom model methods for entity-specific logic

Example of a service object:

```ruby
# app/services/task_manager.rb
class TaskManager
  def self.assign_task(task, user)
    return false unless user.projects.include?(task.project)
    
    task.update(assigned_user_id: user.id)
    TaskMailer.assignment_notification(task, user).deliver_later
    true
  end
end
```

### 2. **Use Scopes for Common Queries**

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  # Scopes
  scope :overdue, -> { where('due_date < ?', Date.current) }
  scope :by_priority, ->(priority) { where(priority: priority) }
  scope :upcoming, -> { where('due_date BETWEEN ? AND ?', Date.current, 7.days.from_now) }
end

# Usage
Task.overdue.by_priority('high')
```

### 3. **Use Callbacks Carefully**

```ruby
# app/models/project.rb
class Project < ApplicationRecord
  after_create :notify_admin
  before_destroy :check_deletable
  
  private
  
  def notify_admin
    AdminMailer.new_project_notification(self).deliver_later
  end
  
  def check_deletable
    throw(:abort) if tasks.in_progress.exists?
  end
end
```

### 4. **Consider Using STI for Similar Models**

Single Table Inheritance can be useful for models that share most attributes:

```ruby
# Base model
class Document < ApplicationRecord
  # Shared attributes and methods
end

# Specialized models
class Invoice < Document
  # Invoice-specific methods
end

class Report < Document
  # Report-specific methods
end
```

### 5. **Leverage Polymorphic Associations**

For relationships where a model can belong to multiple types of models:

```ruby
# app/models/attachment.rb
class Attachment < ApplicationRecord
  belongs_to :attachable, polymorphic: true
end

# app/models/task.rb
class Task < ApplicationRecord
  has_many :attachments, as: :attachable
end

# app/models/project.rb
class Project < ApplicationRecord
  has_many :attachments, as: :attachable
end
```

This creates a flexible structure where attachments can belong to either tasks or projects.
