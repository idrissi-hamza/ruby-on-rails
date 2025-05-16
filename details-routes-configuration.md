# Rails Routes Configuration Explained

## Overview

<strong>Click to view the complete routes configuration file</strong>

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

</details>

This configuration file defines the API routes for a Ruby on Rails application. It organizes endpoints into a versioned API structure with resources for users, projects, tasks, and comments.

## Different Routing Syntaxes

<strong>Understanding the two different routing syntaxes</strong>

Rails offers two main approaches to defining routes, and this file demonstrates both:

### 1. Individual Route Definition

```ruby
post '/login', to: 'authentication#login'
```

This is the "manual" or "individual" route definition syntax:

- You explicitly specify the HTTP verb (`post`)
- You define the exact path (`'/login'`)
- You map it directly to a controller action (`'authentication#login'`)

This approach gives you precise control over a single route and is ideal for:

- Custom, non-RESTful actions
- Authentication endpoints
- Simple actions that don't fit the standard CRUD pattern
- Routes with unique requirements

### 2. Resource-Based Routes

```ruby
resources :users, only: [:index, :show, :update, :destroy]
```

This is the "resource-based" route definition that uses Rails conventions:

- The `resources` helper automatically generates multiple routes following RESTful conventions
- It creates standard CRUD routes without you having to define each one
- In this case, it creates 4 routes (index, show, update, destroy) rather than all 7 standard RESTful routes

The equivalent manual definitions for the `resources :users` line would be:

```ruby
get    '/users',          to: 'users#index'
get    '/users/:id',      to: 'users#show'
patch  '/users/:id',      to: 'users#update'
put    '/users/:id',      to: 'users#update'  # Rails maps both PUT and PATCH to update by default
delete '/users/:id',      to: 'users#destroy'
```

### When to Use Each Syntax

**Use individual route definitions when:**

- You need custom, non-RESTful routes
- You're creating singular actions (like login/logout)
- You want specific URL paths that don't follow RESTful conventions

**Use resource-based routes when:**

- You're building standard CRUD functionality
- You want to follow RESTful conventions
- You need to define many related routes efficiently
- You're working with resources that have standard actions

Rails encourages using resource-based routes when possible, as they promote convention over configuration and ensure your API follows RESTful principles.

</details>

## Detailed Explanation

### API Namespacing

<strong>Namespacing structure explained</strong>

```ruby
namespace :api do
  namespace :v1 do
    # routes defined here
  end
end
```

This structure nests all routes under `/api/v1/`, creating a versioned API. This is a standard practice in API development that allows:

- Clear separation of API endpoints from other application routes
- Future API version changes without breaking existing clients (by adding a `/v2/` namespace later)
- Organization of controllers in corresponding module namespaces (`Api::V1::UsersController`)
</details>

### Authentication Routes

<strong>Authentication routes explained</strong>

```ruby
# Authentication routes
post '/signup', to: 'authentication#signup'
post '/login', to: 'authentication#login'
```

These define two simple endpoints for user authentication:

- `post` indicates these are POST HTTP requests (for submitting data to the server)
- `'/signup'` and `'/login'` are the URL paths that users/applications will send requests to
- `to: 'authentication#signup'` and `to: 'authentication#login'` map these routes to specific controller actions:
  - The `authentication` controller
  - The `signup` and `login` methods/actions within that controller

When users submit data to `/signup`, Rails will call the `signup` method in the `AuthenticationController` to handle the registration process. Similarly, when they submit to `/login`, the `login` method will be called to authenticate them and return a token.

These routes would create:

- `POST /api/v1/signup` - Handled by `Api::V1::AuthenticationController#signup` method
- `POST /api/v1/login` - Handled by `Api::V1::AuthenticationController#login` method

The authentication controller would typically:

- For signup: Create a new user record, validate input, handle password hashing
- For login: Verify credentials and return an authentication token (likely JWT)
</details>

### User Routes

<strong>User routes explained</strong>

```ruby
# User routes
resources :users, only: [:index, :show, :update, :destroy]
```

This creates standard RESTful routes for user management, restricted to only these actions:

- `GET /api/v1/users` - List all users
- `GET /api/v1/users/:id` - Get a specific user
- `PATCH/PUT /api/v1/users/:id` - Update a user
- `DELETE /api/v1/users/:id` - Delete a user

The `only:` option excludes the `new` and `edit` actions, which are typically not needed in APIs since they usually serve HTML forms in traditional Rails applications.

</details>

### Nested Project & Task Structure

<strong>Project and task hierarchy explained</strong>

```ruby
# Project routes
resources :projects do
  # Nested task routes
  resources :tasks, shallow: true do
    # Nested comment routes
    resources :comments, only: [:index, :create]
  end
end
```

This creates a hierarchical relationship between projects, tasks, and comments:

**Projects** - Full RESTful routes:

- `GET /api/v1/projects` - List all projects
- `POST /api/v1/projects` - Create a project
- `GET /api/v1/projects/:id` - Show a project
- `PATCH/PUT /api/v1/projects/:id` - Update a project
- `DELETE /api/v1/projects/:id` - Delete a project

**Tasks** - Nested within projects with the `shallow: true` option:

- Collection routes (maintain the parent relationship):
  - `GET /api/v1/projects/:project_id/tasks` - List tasks for a project
  - `POST /api/v1/projects/:project_id/tasks` - Create a task for a project
- Member routes (shortened to exclude the parent):
  - `GET /api/v1/tasks/:id` - Show a task
  - `PATCH/PUT /api/v1/tasks/:id` - Update a task
  - `DELETE /api/v1/tasks/:id` - Delete a task

**Comments** - Nested within tasks, limited to index and create:

- `GET /api/v1/tasks/:task_id/comments` - List comments for a task
- `POST /api/v1/tasks/:task_id/comments` - Create a comment for a task

The `shallow: true` option is a powerful feature that keeps URLs concise while maintaining the hierarchical relationship where it matters. For collection routes, the parent ID is necessary to scope the collection (e.g., "tasks for project X"). For member routes, once you have the task ID, you don't need the project ID anymore since tasks have a unique ID across all projects.

</details>

### Additional Direct Routes

<strong>Direct routes explained</strong>

```ruby
# Direct routes for tasks and comments
resources :tasks, only: [:show, :update, :destroy]
resources :comments, only: [:destroy]
```

These define additional non-nested access points:

**Tasks**:

- `GET /api/v1/tasks/:id` - Show a task directly
- `PATCH/PUT /api/v1/tasks/:id` - Update a task directly
- `DELETE /api/v1/tasks/:id` - Delete a task directly

**Comments**:

- `DELETE /api/v1/comments/:id` - Delete a comment directly

These routes complement the nested routes and provide direct access to resources when the parent context isn't needed.

</details>

## Route Patterns & Design Decisions

<strong>View design principles and patterns</strong>

### RESTful Design

The routes follow REST conventions, organizing resources into standard CRUD operations, which promotes:

- Consistent, predictable API behavior
- Stateless interactions
- Clear separation of concerns

### Resource Relationships

The configuration expresses natural data relationships:

- Projects contain tasks
- Tasks contain comments
- Each resource has appropriate direct access points

### Shallow Nesting

The use of `shallow: true` demonstrates an advanced Rails routing technique that:

- Maintains parent-child relationships where they matter (creating/listing)
- Avoids unnecessarily long URLs for individual resource operations
- Improves API usability and readability

### Versioning Strategy

The `/api/v1/` namespace indicates a forward-thinking approach to API design, anticipating future versions and changes while maintaining backward compatibility.

</details>

## Generated Routes Summary

<strong>Complete table of all routes generated by this configuration</strong>

| HTTP Verb | Path                               | Controller#Action            | Purpose                       |
| --------- | ---------------------------------- | ---------------------------- | ----------------------------- |
| POST      | /api/v1/signup                     | api/v1/authentication#signup | Register a new user           |
| POST      | /api/v1/login                      | api/v1/authentication#login  | Authenticate user & get token |
| GET       | /api/v1/users                      | api/v1/users#index           | List all users                |
| GET       | /api/v1/users/:id                  | api/v1/users#show            | Show a specific user          |
| PATCH/PUT | /api/v1/users/:id                  | api/v1/users#update          | Update a user                 |
| DELETE    | /api/v1/users/:id                  | api/v1/users#destroy         | Delete a user                 |
| GET       | /api/v1/projects                   | api/v1/projects#index        | List all projects             |
| POST      | /api/v1/projects                   | api/v1/projects#create       | Create a new project          |
| GET       | /api/v1/projects/:id               | api/v1/projects#show         | Show a project                |
| PATCH/PUT | /api/v1/projects/:id               | api/v1/projects#update       | Update a project              |
| DELETE    | /api/v1/projects/:id               | api/v1/projects#destroy      | Delete a project              |
| GET       | /api/v1/projects/:project_id/tasks | api/v1/tasks#index           | List tasks for a project      |
| POST      | /api/v1/projects/:project_id/tasks | api/v1/tasks#create          | Create a task in a project    |
| GET       | /api/v1/tasks/:id                  | api/v1/tasks#show            | Show a task                   |
| PATCH/PUT | /api/v1/tasks/:id                  | api/v1/tasks#update          | Update a task                 |
| DELETE    | /api/v1/tasks/:id                  | api/v1/tasks#destroy         | Delete a task                 |
| GET       | /api/v1/tasks/:task_id/comments    | api/v1/comments#index        | List comments for a task      |
| POST      | /api/v1/tasks/:task_id/comments    | api/v1/comments#create       | Create a comment on a task    |
| DELETE    | /api/v1/comments/:id               | api/v1/comments#destroy      | Delete a comment              |

</details>

## Key Concepts

<strong>Important Rails routing concepts used in this file</strong>

### The `resources` Helper

The `resources` helper automatically generates routes following REST conventions. It creates up to seven standard actions:

- `index` - List all resources
- `create` - Create a new resource
- `new` - Display a form for creating a resource
- `edit` - Display a form for editing a resource
- `show` - Display a specific resource
- `update` - Update a specific resource
- `destroy` - Delete a specific resource

### Limiting Routes with `only:`

The `only:` option restricts which routes are generated. For example:

```ruby
resources :users, only: [:index, :show]
```

This would only create the index and show routes, and none of the others.

### Nested Resources

Nesting resources expresses parent-child relationships. For example:

```ruby
resources :projects do
  resources :tasks
end
```

This creates routes like `/projects/1/tasks/2`, indicating "task 2 of project 1".

### Shallow Nesting

The `shallow: true` option makes routes for individual resources (show, update, destroy) unnested, while keeping collection routes (index, create) nested. This creates cleaner URLs while maintaining relationships.

### Namespaces

Namespaces group routes under a common URL prefix and controller namespace. This is useful for versioning APIs or organizing admin interfaces.

</details>

[Return to rails api cheat sheet](rails-api-cheat-sheet.md)
