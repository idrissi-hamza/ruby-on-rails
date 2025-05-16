# Related Controllers in the API

Here are the other controllers:

## AuthenticationController

[skip_before_action](skip-before-action-rails.md)

```ruby
# app/controllers/api/v1/authentication_controller.rb
module Api
  module V1
    class AuthenticationController < ApplicationController
      skip_before_action :authenticate_user, only: [:signup, :login]

      # POST /api/v1/signup
      def signup
        @user = User.new(user_params)
        if @user.save
          token = generate_token(@user)
          render json: { token: token, user: @user }, status: :created
        else
          render json: { errors: @user.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # POST /api/v1/login
      def login
        @user = User.find_by(email: params[:email])
        if @user && @user.authenticate(params[:password])
          token = generate_token(@user)
          render json: { token: token, user: @user }
        else
          render json: { error: 'Invalid credentials' }, status: :unauthorized
        end
      end

      private

      def user_params
        params.permit(:name, :email, :password, :password_confirmation)
      end

      def generate_token(user)
        # JWT token generation logic
        # payload = { user_id: user.id, exp: 24.hours.from_now.to_i }
        # JWT.encode(payload, Rails.application.credentials.secret_key_base)
      end
    end
  end
end
```

## UsersController

```ruby
# app/controllers/api/v1/users_controller.rb
module Api
  module V1
    class UsersController < ApplicationController
      before_action :set_user, only: [:show, :update, :destroy]
      before_action :authorize_user, only: [:update, :destroy]

      # GET /api/v1/users
      def index
        @users = User.all
        render json: @users
      end

      # GET /api/v1/users/:id
      def show
        render json: @user
      end

      # PATCH/PUT /api/v1/users/:id
      def update
        if @user.update(user_params)
          render json: @user
        else
          render json: { errors: @user.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # DELETE /api/v1/users/:id
      def destroy
        @user.destroy
        head :no_content
      end

      private

      def set_user
        @user = User.find(params[:id])
      end

      def user_params
        params.require(:user).permit(:name, :email, :password, :password_confirmation)
      end

      def authorize_user
        unless @user.id == current_user.id
          render json: { error: 'Unauthorized' }, status: :unauthorized
        end
      end
    end
  end
end
```

## TasksController

```ruby
# app/controllers/api/v1/tasks_controller.rb
module Api
  module V1
    class TasksController < ApplicationController
      before_action :authenticate_user
      before_action :set_project, only: [:index, :create]
      before_action :set_task, only: [:show, :update, :destroy]
      before_action :authorize_user, only: [:update, :destroy]

      # GET /api/v1/projects/:project_id/tasks
      def index
        @tasks = @project.tasks
        render json: @tasks
      end

      # GET /api/v1/tasks/:id
      def show
        render json: @task
      end

      # POST /api/v1/projects/:project_id/tasks
      def create
        @task = @project.tasks.build(task_params)
        if @task.save
          render json: @task, status: :created
        else
          render json: { errors: @task.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # PATCH/PUT /api/v1/tasks/:id
      def update
        if @task.update(task_params)
          render json: @task
        else
          render json: { errors: @task.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # DELETE /api/v1/tasks/:id
      def destroy
        @task.destroy
        head :no_content
      end

      private

      def set_project
        @project = Project.find(params[:project_id])
        unless @project.user_id == current_user.id
          render json: { error: 'Unauthorized' }, status: :unauthorized
        end
      end

      def set_task
        @task = Task.find(params[:id])
      end

      def task_params
        params.require(:task).permit(:title, :description, :due_date, :status, :priority)
      end

      def authorize_user
        unless @task.project.user_id == current_user.id
          render json: { error: 'Unauthorized' }, status: :unauthorized
        end
      end
    end
  end
end
```

## CommentsController

```ruby
# app/controllers/api/v1/comments_controller.rb
module Api
  module V1
    class CommentsController < ApplicationController
      before_action :authenticate_user
      before_action :set_task, only: [:index, :create]
      before_action :set_comment, only: [:destroy]
      before_action :authorize_user, only: [:destroy]

      # GET /api/v1/tasks/:task_id/comments
      def index
        @comments = @task.comments
        render json: @comments
      end

      # POST /api/v1/tasks/:task_id/comments
      def create
        @comment = @task.comments.build(comment_params)
        @comment.user = current_user

        if @comment.save
          render json: @comment, status: :created
        else
          render json: { errors: @comment.errors.full_messages }, status: :unprocessable_entity
        end
      end

      # DELETE /api/v1/comments/:id
      def destroy
        @comment.destroy
        head :no_content
      end

      private

      def set_task
        @task = Task.find(params[:task_id])
        project = @task.project
        unless project.user_id == current_user.id
          render json: { error: 'Unauthorized' }, status: :unauthorized
        end
      end

      def set_comment
        @comment = Comment.find(params[:id])
      end

      def comment_params
        params.require(:comment).permit(:content)
      end

      def authorize_user
        unless @comment.user_id == current_user.id
          render json: { error: 'Unauthorized' }, status: :unauthorized
        end
      end
    end
  end
end
```

## ApplicationController

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::API
  before_action :authenticate_user

  private

  def authenticate_user
    header = request.headers['Authorization']
    token = header.split(' ').last if header

    begin
      decoded = JWT.decode(token, Rails.application.credentials.secret_key_base, true, algorithm: 'HS256')
      @current_user_id = decoded[0]['user_id']
    rescue JWT::DecodeError
      render json: { error: 'Invalid token' }, status: :unauthorized
    end
  end

  def current_user
    @current_user ||= User.find(@current_user_id)
  end
end
```

## Model Relationships

Based on these controllers, the model relationships would likely be:

1. **User**:

   - has_many :projects
   - has_many :comments

2. **Project**:

   - belongs_to :user
   - has_many :tasks

3. **Task**:

   - belongs_to :project
   - has_many :comments

4. **Comment**:
   - belongs_to :task
   - belongs_to :user

## Authentication & Authorization Flow

All controllers follow a similar authentication and authorization pattern:

1. **Authentication** happens via JWT tokens in the `ApplicationController`
2. **Resource Loading** happens in the appropriate `set_X` methods
3. **Authorization** is implemented with owner checks through the `authorize_user` methods

## Security Considerations

Key security aspects implemented across controllers:

1. Token-based authentication for all actions except signup/login
2. Resource ownership verification for all modification operations
3. Strong parameters to prevent mass assignment vulnerabilities
4. Proper error responses with appropriate status codes
5. Authorization checks at different hierarchy levels (project, task, comment)

   [Return to rails api cheat sheet](rails-api-cheat-sheet.md)
