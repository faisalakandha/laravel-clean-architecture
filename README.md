Let's walk through a simple example of implementing **Clean Architecture** in a **Laravel** application. We'll keep things minimal, so you can focus on the core concepts of Clean Architecture while still using Laravel's conventions.

### Scenario:
We'll build a simple **Task Management** system, where users can:
1. **Create tasks**
2. **Mark tasks as complete**

We'll structure the app using **Clean Architecture**, with the following layers:
- **Entities**: Core business logic (e.g., the Task model).
- **Use Cases (Interactors)**: Application logic that coordinates use cases (e.g., creating a task, completing a task).
- **Interface Adapters**: Laravel controllers that communicate with the user interface.
- **Frameworks & Drivers**: External systems like the database (e.g., Eloquent models, Laravel's database layer).

### Step 1: Define the Directory Structure

We will structure the Laravel application to reflect Clean Architecture principles:
```
app/
├── Core/
│   ├── Entities/
│   │   └── Task.php          # Core entity
│   ├── UseCases/
│   │   ├── CreateTask.php    # Use case for creating tasks
│   │   └── CompleteTask.php  # Use case for completing tasks
│   └── Repositories/
│       └── TaskRepository.php # Interface for task persistence
├── Infrastructure/
│   └── Persistence/
│       └── EloquentTaskRepository.php # Eloquent implementation of TaskRepository
└── Http/
    └── Controllers/
        └── TaskController.php  # Controller that interacts with use cases
```

---

### Step 2: Define the Core Business Logic (Entities)

In **Clean Architecture**, entities represent core business logic. For this example, our **Task** entity will be a simple class.

#### **Task.php** (Core/Entities/Task.php)

```php
<?php

namespace App\Core\Entities;

class Task
{
    private $id;
    private $title;
    private $description;
    private $status;

    public function __construct($id, $title, $description, $status)
    {
        $this->id = $id;
        $this->title = $title;
        $this->description = $description;
        $this->status = $status;
    }

    public function getId()
    {
        return $this->id;
    }

    public function getTitle()
    {
        return $this->title;
    }

    public function getDescription()
    {
        return $this->description;
    }

    public function getStatus()
    {
        return $this->status;
    }

    public function markComplete()
    {
        $this->status = 'completed';
    }
}
```

---

### Step 3: Define Use Cases (Application Logic)

In Clean Architecture, use cases represent the specific application logic that handles business processes. Here, we'll create two use cases: **CreateTask** and **CompleteTask**.

#### **CreateTask.php** (Core/UseCases/CreateTask.php)

```php
<?php

namespace App\Core\UseCases;

use App\Core\Entities\Task;
use App\Core\Repositories\TaskRepository;

class CreateTask
{
    private $taskRepository;

    public function __construct(TaskRepository $taskRepository)
    {
        $this->taskRepository = $taskRepository;
    }

    public function execute($title, $description)
    {
        $task = new Task(null, $title, $description, 'pending');
        return $this->taskRepository->save($task);
    }
}
```

#### **CompleteTask.php** (Core/UseCases/CompleteTask.php)

```php
<?php

namespace App\Core\UseCases;

use App\Core\Repositories\TaskRepository;

class CompleteTask
{
    private $taskRepository;

    public function __construct(TaskRepository $taskRepository)
    {
        $this->taskRepository = $taskRepository;
    }

    public function execute($taskId)
    {
        $task = $this->taskRepository->find($taskId);
        $task->markComplete();
        return $this->taskRepository->save($task);
    }
}
```

---

### Step 4: Define Repositories (Interface for Persistence)

Now we define a **TaskRepository** interface that the outer layer (Laravel Eloquent) will implement.

#### **TaskRepository.php** (Core/Repositories/TaskRepository.php)

```php
<?php

namespace App\Core\Repositories;

use App\Core\Entities\Task;

interface TaskRepository
{
    public function save(Task $task);

    public function find($taskId);
}
```

---

### Step 5: Implement the Repository (Eloquent Model)

Now, in the **Infrastructure Layer**, we implement the repository using Laravel’s **Eloquent** ORM.

#### **EloquentTaskRepository.php** (Infrastructure/Persistence/EloquentTaskRepository.php)

```php
<?php

namespace App\Infrastructure\Persistence;

use App\Core\Entities\Task;
use App\Core\Repositories\TaskRepository;
use App\Models\Task as TaskModel;

class EloquentTaskRepository implements TaskRepository
{
    public function save(Task $task)
    {
        $taskModel = new TaskModel();
        $taskModel->title = $task->getTitle();
        $taskModel->description = $task->getDescription();
        $taskModel->status = $task->getStatus();

        if ($task->getId()) {
            $taskModel = TaskModel::find($task->getId());
            $taskModel->update([
                'status' => $task->getStatus()
            ]);
        } else {
            $taskModel->save();
        }

        return $taskModel; // Return Eloquent model or core task entity
    }

    public function find($taskId)
    {
        $taskModel = TaskModel::find($taskId);
        return new Task($taskModel->id, $taskModel->title, $taskModel->description, $taskModel->status);
    }
}
```

---

### Step 6: Define the Controller (Interface Adapter)

In the **Interface Adapters** layer, we create a controller to handle HTTP requests and map them to use cases.

#### **TaskController.php** (Http/Controllers/TaskController.php)

```php
<?php

namespace App\Http\Controllers;

use App\Core\UseCases\CreateTask;
use App\Core\UseCases\CompleteTask;
use Illuminate\Http\Request;

class TaskController extends Controller
{
    private $createTask;
    private $completeTask;

    public function __construct(CreateTask $createTask, CompleteTask $completeTask)
    {
        $this->createTask = $createTask;
        $this->completeTask = $completeTask;
    }

    public function create(Request $request)
    {
        $task = $this->createTask->execute($request->title, $request->description);
        return response()->json($task);
    }

    public function complete($taskId)
    {
        $task = $this->completeTask->execute($taskId);
        return response()->json($task);
    }
}
```

---

### Step 7: Register Dependencies (Service Container)

Now we need to bind the interfaces to the concrete implementations in the **Laravel service container**. We’ll use **dependency injection** to wire everything together.

#### **AppServiceProvider.php** (App/Providers/AppServiceProvider.php)

```php
<?php

namespace App\Providers;

use App\Core\Repositories\TaskRepository;
use App\Infrastructure\Persistence\EloquentTaskRepository;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(TaskRepository::class, EloquentTaskRepository::class);
    }

    public function boot()
    {
        //
    }
}
```

---

### Step 8: Define Routes

Finally, we set up routes to map HTTP requests to controller actions.

#### **web.php** (routes/web.php)

```php
use App\Http\Controllers\TaskController;

Route::post('/tasks', [TaskController::class, 'create']);
Route::post('/tasks/{id}/complete', [TaskController::class, 'complete']);
```

---

### Step 9: Test the Application

Now, you can test the application by using Postman or any HTTP client to send requests to the endpoints.

- **Create Task**: `POST /tasks`
  - Body: `{"title": "Task 1", "description": "Description of task 1"}`
  
- **Complete Task**: `POST /tasks/{id}/complete`

---

### Summary of Clean Architecture in Laravel:
1. **Entities**: Represent core business logic (Task model).
2. **Use Cases**: Encapsulate application logic (CreateTask, CompleteTask).
3. **Repositories**: Define abstractions for persistence (TaskRepository).
4. **Controllers**: Act as interface adapters to map HTTP requests to use cases.
5. **Service Container**: Bind interfaces to concrete implementations (EloquentTaskRepository).

This structure ensures that your core business logic (task creation, completion) is decoupled from infrastructure concerns like the database and frameworks, making it easier to test and maintain. You can swap out the database, change business logic, or add new features without affecting the core of the application.
