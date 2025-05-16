# API Endpoints and Query Implementation in Rails

When building Rails API endpoints, handling query parameters is essential for filtering, sorting, and pagination. This guide explains how to implement query functionality in your Rails API endpoints.

## Basic Query Parameter Handling

In your controller actions, you can access query parameters through the `params` hash:

```ruby
# app/controllers/api/v1/tasks_controller.rb
module Api
  module V1
    class TasksController < ApplicationController
      # GET /api/v1/tasks?status=in_progress&priority=high
      def index
        @tasks = Task.all
        
        # Filter by status if provided
        @tasks = @tasks.where(status: params[:status]) if params[:status].present?
        
        # Filter by priority if provided
        @tasks = @tasks.where(priority: params[:priority]) if params[:priority].present?
        
        render json: @tasks
      end
    end
  end
end
```

This simple approach works well for basic filtering but can become unwieldy as your API grows in complexity.

## Advanced Query Implementation

For more complex APIs, you can implement a query object pattern:

```ruby
# app/queries/task_query.rb
class TaskQuery
  attr_reader :relation

  def initialize(relation = Task.all)
    @relation = relation
  end
  
  def call(params)
    scoped = relation
    scoped = filter_by_status(scoped, params[:status]) if params[:status].present?
    scoped = filter_by_priority(scoped, params[:priority]) if params[:priority].present?
    scoped = filter_by_due_date(scoped, params[:due_before], params[:due_after]) if params[:due_before].present? || params[:due_after].present?
    scoped = search(scoped, params[:search]) if params[:search].present?
    scoped = sort(scoped, params[:sort_by], params[:sort_direction]) if params[:sort_by].present?
    scoped
  end
  
  private
  
  def filter_by_status(scoped, status)
    scoped.where(status: status)
  end
  
  def filter_by_priority(scoped, priority)
    scoped.where(priority: priority)
  end
  
  def filter_by_due_date(scoped, due_before, due_after)
    if due_before.present? && due_after.present?
      scoped.where('due_date BETWEEN ? AND ?', due_after, due_before)
    elsif due_before.present?
      scoped.where('due_date <= ?', due_before)
    elsif due_after.present?
      scoped.where('due_date >= ?', due_after)
    else
      scoped
    end
  end
  
  def search(scoped, term)
    scoped.where('title ILIKE ? OR description ILIKE ?', "%#{term}%", "%#{term}%")
  end
  
  def sort(scoped, sort_by, direction)
    direction = %w[asc desc].include?(direction.to_s.downcase) ? direction : 'asc'
    
    case sort_by.to_s.downcase
    when 'due_date'
      scoped.order(due_date: direction)
    when 'priority'
      scoped.order(priority: direction)
    when 'created_at'
      scoped.order(created_at: direction)
    else
      scoped.order(created_at: :desc) # Default sorting
    end
  end
end
```

Then in your controller:

```ruby
# app/controllers/api/v1/tasks_controller.rb
def index
  @tasks = TaskQuery.new(current_user.tasks).call(params)
  render json: @tasks
end
```

This pattern has several advantages:
- Keeps controllers slim
- Encapsulates query logic in a single, testable class
- Makes complex queries more maintainable
- Allows for reuse across multiple endpoints

## Pagination Implementation

For pagination, you can use the `kaminari` or `will_paginate` gem:

```ruby
# Gemfile
gem 'kaminari'

# Controller
def index
  @tasks = TaskQuery.new(current_user.tasks).call(params)
  @tasks = @tasks.page(params[:page] || 1).per(params[:per_page] || 10)
  
  render json: @tasks, meta: pagination_meta(@tasks)
end

private

def pagination_meta(paginated_objects)
  {
    current_page: paginated_objects.current_page,
    next_page: paginated_objects.next_page,
    prev_page: paginated_objects.prev_page,
    total_pages: paginated_objects.total_pages,
    total_count: paginated_objects.total_count
  }
end
```

Including pagination metadata helps clients implement UI features like page navigators.

## Handling JSON Response Format

To include pagination metadata in your JSON response, configure Active Model Serializers:

```ruby
# config/initializers/active_model_serializers.rb
ActiveModelSerializers.config.adapter = :json_api

# Or for a custom implementation:
def index
  @tasks = TaskQuery.new(current_user.tasks).call(params)
  @tasks = @tasks.page(params[:page] || 1).per(params[:per_page] || 10)
  
  render json: {
    data: ActiveModel::Serializer::CollectionSerializer.new(
      @tasks, 
      serializer: TaskSerializer
    ),
    meta: {
      pagination: pagination_meta(@tasks),
      filters_applied: filter_params.to_h
    }
  }
end
```

## Search with Multiple Parameters Example

For a complex endpoint like "Get all tasks for a user":

```ruby
# GET /api/v1/tasks?status=in_progress&priority=high&due_after=2023-01-01&search=important&sort_by=due_date&sort_direction=asc
def index
  # Base query - get tasks for current user
  base_query = current_user.tasks.includes(:project)
  
  # Apply all filters and sorting
  @tasks = TaskQuery.new(base_query).call(params)
  
  # Paginate results
  @tasks = @tasks.page(params[:page] || 1).per(params[:per_page] || 10)
  
  # Prepare response with pagination metadata
  render json: {
    tasks: ActiveModel::Serializer::CollectionSerializer.new(
      @tasks, 
      serializer: TaskSerializer
    ),
    meta: pagination_meta(@tasks)
  }
end
```

The `includes(:project)` statement is important for performance, as it eager loads the associated project for each task to avoid N+1 query problems.

## Routes Configuration

Configure your routes to support these query parameters:

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :tasks, only: [:index, :show, :update, :destroy] do
      collection do
        get 'search'  # Optional dedicated search endpoint
      end
    end
  end
end
```

For more complex search functionality, you might want a dedicated search endpoint:

```ruby
# app/controllers/api/v1/tasks_controller.rb
def search
  @tasks = TaskQuery.new(current_user.tasks).call(search_params)
  @tasks = @tasks.page(params[:page] || 1).per(params[:per_page] || 10)
  
  render json: {
    tasks: ActiveModel::Serializer::CollectionSerializer.new(
      @tasks, 
      serializer: TaskSerializer
    ),
    meta: pagination_meta(@tasks)
  }
end

private

def search_params
  params.permit(:status, :priority, :search, :page, :per_page, :sort_by, 
                :sort_direction, :due_before, :due_after)
end
```

## Strong Parameters for Security

Ensure you're using strong parameters to prevent mass assignment vulnerabilities:

```ruby
# For creating/updating tasks
def task_params
  params.require(:task).permit(:title, :description, :status, :priority, :due_date)
end

# For filtering/pagination (separate from resource params)
def filter_params
  params.permit(:status, :priority, :search, :page, :per_page, :sort_by, :sort_direction, :due_before, :due_after)
end
```

This separation makes your code more explicit about which parameters are for filtering vs. which are for creating/updating resources.

## Request Examples

For a complete example, here's how a client would call these endpoints:

```
# Get all tasks
GET /api/v1/tasks

# Get high priority tasks
GET /api/v1/tasks?priority=high

# Get in-progress tasks due in the next week, sorted by due date
GET /api/v1/tasks?status=in_progress&due_after=2023-05-16&due_before=2023-05-23&sort_by=due_date&sort_direction=asc

# Search tasks containing "presentation" in title or description
GET /api/v1/tasks?search=presentation

# Get second page of results, 15 per page
GET /api/v1/tasks?page=2&per_page=15
```

## Advanced Filtering with JSON Parameters

For more complex filtering, you can accept JSON structured parameters:

```ruby
# GET /api/v1/tasks?filter={"status":["in_progress","todo"],"priority":"high"}
def index
  filter_json = JSON.parse(params[:filter]) if params[:filter].present?
  
  @tasks = current_user.tasks
  
  if filter_json.present?
    @tasks = @tasks.where(status: filter_json['status']) if filter_json['status'].present?
    @tasks = @tasks.where(priority: filter_json['priority']) if filter_json['priority'].present?
    # Additional filtering...
  end
  
  render json: @tasks
end
```

## Using Scopes for Common Queries

You can leverage ActiveRecord scopes to make your query objects even cleaner:

```ruby
# app/models/task.rb
class Task < ApplicationRecord
  # Scopes
  scope :with_status, ->(status) { where(status: status) }
  scope :with_priority, ->(priority) { where(priority: priority) }
  scope :due_between, ->(start_date, end_date) { where('due_date BETWEEN ? AND ?', start_date, end_date) }
  scope :search_term, ->(term) { where('title ILIKE ? OR description ILIKE ?', "%#{term}%", "%#{term}%") }
  
  # More scopes...
end

# Then in your query object
def filter_by_status(scoped, status)
  scoped.with_status(status)
end
```

## Handling Complex Filtering Logic

For complex filtering with multiple conditions, logical operators, etc., consider using a gem like Ransack or building a more sophisticated query builder:

```ruby
# Gemfile
gem 'ransack'

# Controller
def index
  @q = current_user.tasks.ransack(params[:q])
  @tasks = @q.result(distinct: true).page(params[:page]).per(params[:per_page])
  
  render json: @tasks
end
```

With Ransack, clients can build complex queries:

```
GET /api/v1/tasks?q[status_eq]=in_progress&q[priority_eq]=high&q[due_date_gteq]=2023-05-01&q[s]=due_date+asc
```

## Performance Considerations

When implementing complex queries, keep these performance tips in mind:

1. **Use proper indexes** for columns you filter and sort by
2. **Eager load associations** that you'll reference in the response
3. **Monitor query performance** in development and production
4. **Cache frequent queries** when appropriate
5. **Limit result sets** with pagination
6. **Use `explain`** to analyze complex queries:

```ruby
Task.where(status: 'in_progress', priority: 'high').order(due_date: :asc).explain
```

This approach gives you a flexible, maintainable way to handle complex query parameters in your Rails API endpoints.
