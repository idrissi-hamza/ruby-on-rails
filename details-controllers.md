# ProjectsController Explanation

## Controller Overview

This `ProjectsController` is a RESTful Rails controller that manages CRUD operations for project resources in a versioned API. It implements security measures to ensure users can only access and modify their own projects, and follows standard Rails conventions for handling requests and responses.

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

## Namespacing Structure

The controller is nested within two modules:
- `Api` - Top-level namespace for all API controllers
- `V1` - Version 1 of the API

This organization corresponds to the route namespacing in `routes.rb` and creates a clear structure for API versioning.

## Before Action Filters

The controller uses three `before_action` callbacks to set up requests:

1. `authenticate_user`: Applied to all actions, ensures the user is logged in before any project operations can be performed. This likely uses a JWT token or similar authentication mechanism.

2. `set_project`: Runs before `show`, `update`, and `destroy` actions to find the project by ID and make it available via `@project`.

3. `authorize_user`: Runs before `update` and `destroy` to verify the current user owns the project, preventing unauthorized modifications of other users' projects.

## CRUD Actions

### Index Action
```ruby
def index
  @projects = current_user.projects
  render json: @projects
end
```
- Retrieves all projects belonging to the current user
- Returns the projects as a JSON array
- Scopes results to current user for security (users can only see their own projects)

### Show Action
```ruby
def show
  render json: @project
end
```
- Displays a specific project (already loaded by `set_project`)
- Returns the project as JSON
- Note that `authorize_user` is not applied, allowing viewing of projects the user doesn't own (may be intentional if projects can be shared)

### Create Action
```ruby
def create
  @project = current_user.projects.build(project_params)
  if @project.save
    render json: @project, status: :created
  else
    render json: { errors: @project.errors.full_messages }, status: :unprocessable_entity
  end
end
```
- Builds a new project associated with the current user
- Uses strong parameters via `project_params` to securely permit only allowed attributes
- Returns the created project with 201 status on success
- Returns validation errors with 422 status on failure

### Update Action
```ruby
def update
  if @project.update(project_params)
    render json: @project
  else
    render json: { errors: @project.errors.full_messages }, status: :unprocessable_entity
  end
end
```
- Updates an existing project (already verified to belong to current user by `authorize_user`)
- Uses the same strong parameters as create
- Returns the updated project on success
- Returns validation errors with 422 status on failure

### Destroy Action
```ruby
def destroy
  @project.destroy
  head :no_content
end
```
- Deletes an existing project (already verified to belong to current user)
- Returns a 204 No Content status with no body on success

## Private Methods

### set_project
```ruby
def set_project
  @project = Project.find(params[:id])
end
```
- Finds a project by its ID from the route params
- Raises `ActiveRecord::RecordNotFound` if project doesn't exist (Rails will convert this to a 404 response)

### project_params
```ruby
def project_params
  params.require(:project).permit(:title, :description, :status)
end
```
- Implements strong parameters to prevent mass assignment vulnerabilities
- Requires a top-level `project` parameter
- Only permits `title`, `description`, and `status` attributes to be set

### authorize_user
```ruby
def authorize_user
  unless @project.user_id == current_user.id
    render json: { error: 'Unauthorized' }, status: :unauthorized
  end
end
```
- Verifies that the current user is the owner of the project
- Returns 401 Unauthorized if a user tries to update or delete someone else's project
- Simple but effective authorization check

## Security Considerations

This controller implements several important security measures:
1. Authentication on all endpoints
2. Scoping of index results to current user only
3. Explicit authorization checks for destructive actions
4. Strong parameters to prevent mass assignment vulnerabilities

## Common Design Patterns

This controller demonstrates several Rails best practices:
- RESTful resource design
- Skinny controllers (business logic would presumably be in models)
- DRY code with before_action callbacks
- Appropriate HTTP status codes
- Consistent error handling
- API versioning through namespaces
- Proper separation of concerns

## Usage Examples

### Creating a project
```http
POST /api/v1/projects
Authorization: Bearer [jwt_token]
Content-Type: application/json

{
  "project": {
    "title": "New Project",
    "description": "This is a project description",
    "status": "active"
  }
}
```

### Updating a project
```http
PUT /api/v1/projects/1
Authorization: Bearer [jwt_token]
Content-Type: application/json

{
  "project": {
    "status": "completed"
  }
}
```

### Listing user's projects
```http
GET /api/v1/projects
Authorization: Bearer [jwt_token]
```
