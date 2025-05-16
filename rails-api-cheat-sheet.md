# Ruby on Rails API Project Guide

## Project Overview: TaskManager API

This guide will walk you through building a complete Task Manager API using Ruby on Rails. The project covers all essential concepts for Rails API development while maintaining a practical, real-world focus.

## Table of Contents

1. [Setup and Configuration](#1-setup-and-configuration)
2. [Ruby Language Fundamentals](#2-ruby-language-fundamentals)
3. [Rails Framework Basics](#3-rails-framework-basics)
4. [API Design and Planning](#4-api-design-and-planning)
5. [Routes and Controllers](#5-routes-and-controllers)
6. [Models and Database Architecture](#6-models-and-database-architecture)
7. [Serialization](#7-serialization)
8. [Authentication and Authorization](#8-authentication-and-authorization)
9. [API Versioning](#9-api-versioning)
10. [Error Handling](#10-error-handling)
11. [Testing](#11-testing)
12. [Performance Optimization](#12-performance-optimization)
13. [Documentation](#13-documentation)
14. [Deployment](#14-deployment)

---

## 1. Setup and Configuration

### Install Dependencies

```bash
# Install Ruby (if not already installed)
# For macOS using Homebrew
brew install ruby

# For Ubuntu/Debian
sudo apt install ruby-full

# Install Rails
gem install rails

# Install PostgreSQL
# macOS
brew install postgresql
# Ubuntu/Debian
sudo apt install postgresql postgresql-contrib
```

### Create a New Rails API Project

```bash
# Create a new Rails API-only application
rails new task_manager_api --api --database=postgresql

# Navigate to the project directory
cd task_manager_api
```

### Configure Database

Edit `config/database.yml` to set up your PostgreSQL connection:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: your_postgres_username
  password: your_postgres_password

development:
  <<: *default
  database: task_manager_api_development

test:
  <<: *default
  database: task_manager_api_test

production:
  <<: *default
  database: task_manager_api_production
  username: task_manager_api
  password: <%= ENV['TASK_MANAGER_API_DATABASE_PASSWORD'] %>
```

### Install Key Gems

Add these gems to your `Gemfile`:

```ruby
# Gemfile
gem 'rack-cors'
gem 'jwt'
gem 'bcrypt'
gem 'active_model_serializers'
gem 'rswag'
gem 'rspec-rails', group: [:development, :test]
gem 'factory_bot_rails', group: [:development, :test]
gem 'faker', group: [:development, :test]
gem 'database_cleaner', group: [:test]
gem 'shoulda-matchers', group: [:test]
```

Run `bundle install` to install the gems.

### Configure CORS

Edit `config/initializers/cors.rb`:

```ruby
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins '*'
    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

Note: For production, replace `'*'` with your frontend domain for security.

---

## 2. Ruby Language Fundamentals

Here's a quick overview of essential Ruby concepts you'll use in Rails API development:

### Variables and Data Types

```ruby
# Variables
name = "John"  # String
age = 30       # Integer
price = 19.99  # Float
is_active = true  # Boolean

# Array
users = ["John", "Jane", "Bob"]

# Hash (similar to JSON objects)
user = {
  name: "John",
  email: "john@example.com",
  age: 30
}
```

### Methods

```ruby
# Method definition
def greet(name)
  "Hello, #{name}!"
end

# Method call
greeting = greet("John")
```

### Classes and Objects

```ruby
class User
  attr_accessor :name, :email

  def initialize(name, email)
    @name = name
    @email = email
  end

  def display_info
    "#{name} (#{email})"
  end
end

user = User.new("John", "john@example.com")
puts user.display_info
```

### Control Flow

```ruby
# Conditional statements
if age >= 18
  "Adult"
elsif age >= 13
  "Teenager"
else
  "Child"
end

# Loops
users.each do |user|
  puts user
end

# Map transformation
user_emails = users.map { |user| user.email }
```

---

## 3. Rails Framework Basics

### MVC Architecture

Rails follows the Model-View-Controller (MVC) pattern:

- **Models**: Handle data and business logic
- **Views**: In API mode, these are JSON responses instead of HTML
- **Controllers**: Handle HTTP requests and responses

### RESTful Conventions

Rails encourages RESTful API design with these standard actions:

- **index**: List all resources
- **show**: Show a single resource
- **create**: Create a new resource
- **update**: Update an existing resource
- **destroy**: Delete a resource

### Directory Structure

Important directories in your Rails API project:

```
app/
  controllers/ - API controllers
  models/ - Database models
  serializers/ - Format JSON responses
config/
  routes.rb - Define API endpoints
db/
  migrations/ - Database structure changes
spec/ - Tests
```

---

## 4. API Design and Planning

Let's plan our TaskManager API resources:

### Resources

1. Users
2. Projects
3. Tasks
4. Comments

### Entity Relationships

- User has many Projects
- Project belongs to User
- Project has many Tasks
- Task belongs to Project
- Task has many Comments
- Comment belongs to Task and User

### API Endpoints

#### Authentication

- POST /api/v1/signup
- POST /api/v1/login

#### Users

- GET /api/v1/users
- GET /api/v1/users/:id
- PUT /api/v1/users/:id
- DELETE /api/v1/users/:id

#### Projects

- GET /api/v1/projects
- GET /api/v1/projects/:id
- POST /api/v1/projects
- PUT /api/v1/projects/:id
- DELETE /api/v1/projects/:id

#### Tasks

- GET /api/v1/projects/:project_id/tasks
- GET /api/v1/tasks/:id
- POST /api/v1/projects/:project_id/tasks
- PUT /api/v1/tasks/:id
- DELETE /api/v1/tasks/:id

#### Comments

- GET /api/v1/tasks/:task_id/comments
- POST /api/v1/tasks/:task_id/comments
- DELETE /api/v1/comments/:id

---

## 5. Routes and Controllers

### Generate Models and Controllers

```bash
# Generate User model and controller
rails g model User name:string email:string password_digest:string
rails g controller Api::V1::Users index show update destroy

# Generate Project model and controller
rails g model Project title:string description:text status:string user:references
rails g controller Api::V1::Projects index show create update destroy

# Generate Task model and controller
rails g model Task title:string description:text due_date:datetime status:string priority:string project:references
rails g controller Api::V1::Tasks index show create update destroy

# Generate Comment model and controller
rails g model Comment content:text user:references task:references
rails g controller Api::V1::Comments index create destroy

# Generate Authentication controller
rails g controller Api::V1::Authentication signup login
```

### Routes Configuration

[Routes Configuration Details](details-routes-configuration.md)

Define your API routes in `config/routes.rb`:

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do

      # Authentication routes
      post '/signup', to: 'authentication#signup'
      post '/login', to: 'authentication#login'

      # User routes
      resources :users, only: [:index, :show, :update, :destroy]

      # Project routes
      resources :projects do
        # Nested task routes
        resources :tasks, shallow: true do
          # Nested comment routes
          resources :comments, only: [:index, :create]
        end
      end

      # Direct routes for tasks and comments
      resources :tasks, only: [:show, :update, :destroy]
      resources :comments, only: [:destroy]
    end
  end
end
```

### Controller Implementation Example

Let's implement the Projects controller as an example:

```ruby
# app/controllers/api/v1/projects_controller.rb
module Api
  module V1
    class ProjectsController < ApplicationController
      before_action :authenticate_user
      before_action :set_project, only: [:show, :update, :destroy]
      before_action :authorize_user, only: [:update, :destroy]

      # GET /api/v1/projects
      def index
        @projects = current_user.projects
        render json: @projects
      end

      # GET /api/v1/projects/:id
      def show
        render json: @project
      end

      # POST /api/v1/projects
      def create
        @project = current_user.projects.build(project_params)

        if @project.save
          render json: @project, status: :created
        else
          render json: { errors: @project.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # PUT /api/v1/projects/:id
      def update
        if @project.update(project_params)
          render json: @project
        else
          render json: { errors: @project.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # DELETE /api/v1/projects/:id
      def destroy
        @project.destroy
        head :no_content
      end

      private

      def set_project
        @project = Project.find(params[:id])
      end

      def project_params
        params.require(:project).permit(:title, :description, :status)
      end

      def authorize_user
        unless @project.user_id == current_user.id
          render json: { error: 'Unauthorized' }, status: :unauthorized
        end
      end
    end
  end
end
```

[Projects controller Details](projects-controller.md)

[Other controllers Details](related-controllers.md)

---

## 6. Models and Database Architecture

### Model Associations

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_secure_password

  has_many :projects, dependent: :destroy
  has_many :comments, dependent: :destroy

  # Add all tasks for a user through their projects
  has_many :tasks, through: :projects

  validates :name, presence: true
  validates :email, presence: true, uniqueness: true, format: { with: URI::MailTo::EMAIL_REGEXP }
end

# app/models/project.rb
class Project < ApplicationRecord
  belongs_to :user
  has_many :tasks, dependent: :destroy

  validates :title, presence: true
  validates :status, inclusion: { in: ['planning', 'in_progress', 'completed'] }
end

# app/models/task.rb
class Task < ApplicationRecord
  belongs_to :project
  has_many :comments, dependent: :destroy

  validates :title, presence: true
  validates :status, inclusion: { in: ['todo', 'in_progress', 'done'] }
  validates :priority, inclusion: { in: ['low', 'medium', 'high'] }

  # Delegate user to project
  delegate :user, to: :project
end

# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :user
  belongs_to :task

  validates :content, presence: true
end
```

### Migrations

Let's enhance our migrations with additional constraints and indexes:

```ruby
# Update the Users migration
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

# Update the Projects migration
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

# Update the Tasks migration
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

Run migrations with:

```bash
rails db:migrate
```

[more details](rails-models-db.md)

---

## 7. Serialization

We'll use Active Model Serializers to format our API responses:

### Install and Configure AMS

First, ensure `active_model_serializers` is in your Gemfile and run `bundle install`.

### Create Serializers

```bash
# Generate serializers for each model
rails g serializer user
rails g serializer project
rails g serializer task
rails g serializer comment
```

### Implement Serializers

```ruby
# app/serializers/user_serializer.rb
class UserSerializer < ActiveModel::Serializer
  attributes :id, :name, :email, :created_at, :updated_at

  has_many :projects
end

# app/serializers/project_serializer.rb
class ProjectSerializer < ActiveModel::Serializer
  attributes :id, :title, :description, :status, :created_at, :updated_at

  belongs_to :user
  has_many :tasks
end

# app/serializers/task_serializer.rb
class TaskSerializer < ActiveModel::Serializer
  attributes :id, :title, :description, :due_date, :status, :priority,
             :created_at, :updated_at

  belongs_to :project
  has_many :comments
end

# app/serializers/comment_serializer.rb
class CommentSerializer < ActiveModel::Serializer
  attributes :id, :content, :created_at, :updated_at

  belongs_to :user
  belongs_to :task
end
```

---

## 8. Authentication and Authorization

We'll implement JWT (JSON Web Token) authentication for our API:

### Authentication Controller

```ruby
# app/controllers/api/v1/authentication_controller.rb
module Api
  module V1
    class AuthenticationController < ApplicationController
      skip_before_action :authenticate_user, only: [:login, :signup]

      # POST /api/v1/signup
      def signup
        user = User.new(user_params)

        if user.save
          token = jwt_encode(user_id: user.id)
          render json: { user: UserSerializer.new(user), token: token }, status: :created
        else
          render json: { errors: user.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # POST /api/v1/login
      def login
        user = User.find_by(email: params[:email])

        if user&.authenticate(params[:password])
          token = jwt_encode(user_id: user.id)
          render json: { user: UserSerializer.new(user), token: token }
        else
          render json: { error: 'Invalid email or password' }, status: :unauthorized
        end
      end

      private

      def user_params
        params.permit(:name, :email, :password, :password_confirmation)
      end
    end
  end
end
```

### JWT Token Handling

Create a JWT handling module in `app/controllers/concerns/json_web_token.rb`:

```ruby
module JsonWebToken
  extend ActiveSupport::Concern

  SECRET_KEY = Rails.application.credentials.secret_key_base

  def jwt_encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET_KEY)
  end

  def jwt_decode(token)
    decoded = JWT.decode(token, SECRET_KEY)[0]
    HashWithIndifferentAccess.new(decoded)
  rescue JWT::DecodeError, JWT::ExpiredSignature
    nil
  end
end
```

### Authentication in Base Controller

Update the ApplicationController to handle authentication:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include JsonWebToken

  before_action :authenticate_user

  private

  def authenticate_user
    header = request.headers['Authorization']
    header = header.split(' ').last if header

    begin
      @decoded = jwt_decode(header)
      @current_user = User.find(@decoded[:user_id])
    rescue ActiveRecord::RecordNotFound
      render json: { error: 'Unauthorized' }, status: :unauthorized
    end
  end

  def current_user
    @current_user
  end
end
```

---

## 9. API Versioning

We're already using namespace-based versioning with `api/v1` prefix. This allows us to create new versions (v2, v3) while maintaining backward compatibility.

### Version Constraints

If you need more sophisticated versioning, you can use header-based versioning:

```ruby
# config/initializers/api_version.rb
module ApiVersion
  class Constraint
    def initialize(version)
      @version = version
    end

    def matches?(request)
      request.headers['Accept'].include?("application/vnd.taskmanager.v#{@version}")
    end
  end
end
```

Then in routes:

```ruby
# config/routes.rb
Rails.application.routes.draw do
  # API v1 routes with header-based version constraint
  scope module: :api, defaults: { format: :json } do
    scope module: :v1, constraints: ApiVersion::Constraint.new(1) do
      # Routes here
    end

    # Future versions
    scope module: :v2, constraints: ApiVersion::Constraint.new(2) do
      # V2 routes here
    end
  end
end
```

---

## 10. Error Handling

### Custom Error Handling

Create an errors module in `app/controllers/concerns/error_handler.rb`:

```ruby
module ErrorHandler
  extend ActiveSupport::Concern

  included do
    rescue_from ActiveRecord::RecordNotFound, with: :not_found
    rescue_from ActiveRecord::RecordInvalid, with: :unprocessable_entity
    rescue_from ActionController::ParameterMissing, with: :bad_request
  end

  private

  def not_found(exception)
    render json: { error: exception.message }, status: :not_found
  end

  def unprocessable_entity(exception)
    render json: { errors: exception.record.errors.full_messages }, status: :unprocessable_entity
  end

  def bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end
end
```

Include this in your ApplicationController:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include JsonWebToken
  include ErrorHandler

  # Rest of the code...
end
```

### Custom Response Format

For consistent API responses, create a response formatter:

```ruby
# app/controllers/concerns/response.rb
module Response
  def json_response(object, status = :ok)
    render json: object, status: status
  end
end
```

Include in ApplicationController:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  include JsonWebToken
  include ErrorHandler
  include Response

  # Rest of the code...
end
```

---

## 11. Testing

### RSpec Setup

Run the following commands to set up RSpec:

```bash
rails generate rspec:install
```

### Test Database Configuration

Update `config/database.yml` for test environment.

### Factory Bot Setup

Create factories for your models:

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    name { Faker::Name.name }
    email { Faker::Internet.unique.email }
    password { 'password123' }
  end
end

# spec/factories/projects.rb
FactoryBot.define do
  factory :project do
    title { Faker::App.name }
    description { Faker::Lorem.paragraph }
    status { ['planning', 'in_progress', 'completed'].sample }
    association :user
  end
end

# spec/factories/tasks.rb
FactoryBot.define do
  factory :task do
    title { Faker::Lorem.sentence }
    description { Faker::Lorem.paragraph }
    due_date { Faker::Date.forward(days: 30) }
    status { ['todo', 'in_progress', 'done'].sample }
    priority { ['low', 'medium', 'high'].sample }
    association :project
  end
end

# spec/factories/comments.rb
FactoryBot.define do
  factory :comment do
    content { Faker::Lorem.paragraph }
    association :user
    association :task
  end
end
```

### Model Tests

Example of a model test:

```ruby
# spec/models/user_spec.rb
require 'rails_helper'

RSpec.describe User, type: :model do
  # Associations
  it { should have_many(:projects).dependent(:destroy) }
  it { should have_many(:comments).dependent(:destroy) }
  it { should have_many(:tasks).through(:projects) }

  # Validations
  it { should validate_presence_of(:name) }
  it { should validate_presence_of(:email) }
  it { should validate_uniqueness_of(:email) }

  # Create a valid user
  let(:valid_user) { build(:user) }

  it "is valid with valid attributes" do
    expect(valid_user).to be_valid
  end

  it "is not valid without a password" do
    user = build(:user, password: nil)
    expect(user).not_to be_valid
  end

  it "hashes the password" do
    user = create(:user, password: "password123")
    expect(user.password_digest).not_to eq "password123"
  end
end
```

### Request Tests

Example of a request test:

```ruby
# spec/requests/api/v1/projects_spec.rb
require 'rails_helper'

RSpec.describe "Api::V1::Projects", type: :request do
  let(:user) { create(:user) }
  let!(:projects) { create_list(:project, 3, user: user) }
  let(:project_id) { projects.first.id }
  let(:headers) { valid_headers }

  # Helper method for authorization headers
  def valid_headers
    {
      "Authorization" => token_generator(user.id),
      "Content-Type" => "application/json"
    }
  end

  # Helper method for token generation
  def token_generator(user_id)
    JsonWebToken.jwt_encode(user_id: user_id)
  end

  describe "GET /api/v1/projects" do
    before { get "/api/v1/projects", headers: headers }

    it "returns projects" do
      expect(json).not_to be_empty
      expect(json.size).to eq(3)
    end

    it "returns status code 200" do
      expect(response).to have_http_status(200)
    end
  end

  describe "GET /api/v1/projects/:id" do
    before { get "/api/v1/projects/#{project_id}", headers: headers }

    context "when the record exists" do
      it "returns the project" do
        expect(json).not_to be_empty
        expect(json['id']).to eq(project_id)
      end

      it "returns status code 200" do
        expect(response).to have_http_status(200)
      end
    end

    context "when the record does not exist" do
      let(:project_id) { 100 }

      it "returns status code 404" do
        expect(response).to have_http_status(404)
      end

      it "returns a not found message" do
        expect(response.body).to match(/Couldn't find Project/)
      end
    end
  end

  # Additional tests for POST, PUT, DELETE...
end
```

---

## 12. Performance Optimization

### Database Optimization

#### Indexes

We've already added indexes to our migrations. Here are a few more considerations:

1. Add composite indexes for frequently queried combinations
2. Add database-level constraints for data integrity

#### Query Optimization

##### N+1 Query Problem

Use eager loading to avoid N+1 queries:

```ruby
# Bad - causes N+1 queries
def index
  @projects = current_user.projects
  render json: @projects
end

# Good - eager loads tasks
def index
  @projects = current_user.projects.includes(:tasks)
  render json: @projects
end
```

### Caching

Add caching for frequently accessed data:

```ruby
# config/environments/production.rb
config.cache_store = :redis_cache_store, { url: ENV['REDIS_URL'] }

# In controllers
def show
  @project = Rails.cache.fetch("project_#{params[:id]}", expires_in: 1.hour) do
    current_user.projects.find(params[:id])
  end
  render json: @project
end
```

### Pagination

Add the `kaminari` or `will_paginate` gem to implement pagination:

```ruby
# Gemfile
gem 'kaminari'

# In controllers
def index
  @projects = current_user.projects.page(params[:page]).per(params[:per_page] || 10)
  render json: @projects
end
```

---

## 13. Documentation

### RSwag for API Documentation

With RSwag installed, you can generate Swagger documentation:

```bash
rails g rswag:api:install
rails g rswag:ui:install
```

### Create Swagger Specs

```ruby
# spec/swagger/api/v1/projects_controller_spec.rb
require 'swagger_helper'

describe 'Projects API' do
  path '/api/v1/projects' do
    get 'Retrieves all projects' do
      tags 'Projects'
      security [Bearer: []]
      produces 'application/json'

      response '200', 'projects found' do
        schema type: :array,
          items: {
            type: :object,
            properties: {
              id: { type: :integer },
              title: { type: :string },
              description: { type: :string },
              status: { type: :string },
              created_at: { type: :string, format: :datetime },
              updated_at: { type: :string, format: :datetime }
            },
            required: ['id', 'title', 'status']
          }

        run_test!
      end

      response '401', 'unauthorized' do
        schema type: :object,
          properties: {
            error: { type: :string }
          }

        run_test!
      end
    end

    # More operations...
  end
end
```

### Generate Swagger JSON

```bash
rails rswag:specs:swaggerize
```

The Swagger UI will be available at `/api-docs`.

---

## 14. Deployment

### Preparing for Production

#### Environment Variables

Use the `dotenv-rails` gem for environment variables:

```ruby
# Gemfile
gem 'dotenv-rails'
```

Create a `.env` file:

```
DATABASE_URL=postgres://user:password@localhost/task_manager_production
JWT_SECRET_KEY=your_secret_key
RAILS_MAX_THREADS=5
```

#### Procfile

Create a `Procfile` for services:

```
web: bundle exec puma -C config/puma.rb
release: bundle exec rails db:migrate
```

### Heroku Deployment

```bash
# Install Heroku CLI
brew install heroku/brew/heroku

# Login to Heroku
heroku login

# Create Heroku app
heroku create task-manager-api

# Add PostgreSQL
heroku addons:create heroku-postgresql:hobby-dev

# Set environment variables
heroku config:set RAILS_MASTER_KEY=$(cat config/master.key)

# Deploy
git push heroku main

# Run migrations
heroku run rails db:migrate
```

### Other Deployment Options

#### AWS Elastic Beanstalk

1. Create Elastic Beanstalk environment
2. Set up RDS for database
3. Deploy with EB CLI or AWS console

#### Docker Deployment

Create a `Dockerfile`:

```dockerfile
FROM ruby:3.2.0

RUN apt-get update -qq && apt-get install -y nodejs postgresql-client
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .

EXPOSE 3000
CMD ["rails", "server", "-b", "0.0.0.0"]
```

Create `docker-compose.yml`:

```yaml
version: '3'
services:
  db:
    image: postgres:14
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD: password

  web:
    build: .
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -b '0.0.0.0'"
    volumes:
      - .:/app
    ports:
      - '3000:3000'
    depends_on:
      - db
    environment:
      DATABASE_URL: postgres://postgres:password@db/task_manager_production

volumes:
  postgres_data:
```

---

## Next Steps

Congratulations! You now have a comprehensive understanding of Ruby on Rails API development. To continue your learning journey:

1. **Advanced Authentication**: Implement OAuth, social login, or two-factor authentication
2. **Background Jobs**: Add Sidekiq or Delayed Job for handling long-running processes
3. **Real-time Features**: Implement ActionCable for WebSockets
4. **Microservices**: Split your API into smaller, specialized services
5. **API Gateways**: Add an API gateway for rate limiting and analytics
6. **Monitoring**: Implement logging and monitoring solutions

Remember that this guide covers the essentials, but Rails is a deep framework with many additional features. Keep exploring the [Rails Guides](https://guides.rubyonrails.org/) and experimenting with new patterns and techniques.

Happy coding!
