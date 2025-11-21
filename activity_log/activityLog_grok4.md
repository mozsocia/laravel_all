https://grok.com/share/c2hhcmQtNQ_8311aa80-bae4-43fc-ad0d-a9c606ac4544

### 🔴 i want to log all activity of my laravel project like spatie/laravel-activitylog but without using any package, please guide me


### Step 1: Understand the Goal
You're looking to implement activity logging in your Laravel project similar to `spatie/laravel-activitylog`. This typically involves tracking changes to Eloquent models (e.g., created, updated, deleted) and storing details like what happened, who did it, and the affected data. We'll do this manually without third-party packages by leveraging Laravel's built-in features like Eloquent events, migrations, and polymorphic relationships.

Key features we'll cover:
- A database table to store logs.
- A model for logs.
- A reusable trait to attach logging to any Eloquent model.
- Automatic logging on model events (created, updated, deleted).
- Recording the "causer" (e.g., the authenticated user).
- Storing additional properties (e.g., old/new values for updates).

Assumptions:
- You're using Laravel 8+ (for modern features like casts).
- You have authentication set up (e.g., via Laravel's `auth()` helper).
- Logs will be stored in a table called `activity_logs`.

### Step 2: Create the Migration for the Activity Logs Table
Run the following Artisan command to create a migration:

```
php artisan make:migration create_activity_logs_table
```

Edit the migration file (e.g., `database/migrations/xxxx_xx_xx_create_activity_logs_table.php`) with this code:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('activity_logs', function (Blueprint $table) {
            $table->id();
            $table->string('log_name')->nullable(); // e.g., 'default' or custom group
            $table->string('description'); // e.g., 'User created'
            $table->morphs('subject'); // Polymorphic: subject_id and subject_type (e.g., App\Models\User)
            $table->morphs('causer')->nullable(); // Polymorphic: causer_id and causer_type (e.g., the user who performed the action)
            $table->json('properties')->nullable(); // JSON for old/new values, etc.
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('activity_logs');
    }
};
```

Run the migration:
```
php artisan migrate
```

### Step 3: Create the ActivityLog Model
Create a model for the logs:

```
php artisan make:model ActivityLog
```

Edit `app/Models/ActivityLog.php`:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class ActivityLog extends Model
{
    protected $table = 'activity_logs';

    protected $fillable = [
        'log_name',
        'description',
        'subject_id',
        'subject_type',
        'causer_id',
        'causer_type',
        'properties',
    ];

    protected $casts = [
        'properties' => 'array', // Automatically cast JSON to array
    ];

    public function subject(): MorphTo
    {
        return $this->morphTo();
    }

    public function causer(): MorphTo
    {
        return $this->morphTo();
    }
}
```

### Step 4: Create a LogsActivity Trait
This trait will be attached to any model you want to log. Create a new file: `app/Traits/LogsActivity.php`

```php
<?php

namespace App\Traits;

use App\Models\ActivityLog;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Auth;

trait LogsActivity
{
    protected static function bootLogsActivity()
    {
        // Listen to created event
        static::created(function (Model $model) {
            $model->logActivity('created');
        });

        // Listen to updated event
        static::updated(function (Model $model) {
            if (!$model->wasRecentlyCreated) { // Avoid logging 'created' as 'updated'
                $model->logActivity('updated', $model->getChanges());
            }
        });

        // Listen to deleted event
        static::deleted(function (Model $model) {
            $model->logActivity('deleted');
        });

        // You can add more events like 'restored' for soft deletes if needed
    }

    protected function logActivity(string $event, array $changes = []): void
    {
        $description = $this->getDescriptionForEvent($event);

        $log = [
            'log_name' => $this->getLogName(),
            'description' => $description,
            'subject_id' => $this->getKey(),
            'subject_type' => get_class($this),
            'causer_id' => Auth::id(), // Assuming user is authenticated; adjust if needed
            'causer_type' => Auth::user() ? get_class(Auth::user()) : null,
            'properties' => $this->getPropertiesForEvent($event, $changes),
        ];

        ActivityLog::create($log);
    }

    // Customizable methods (override in your model if needed)
    protected function getLogName(): string
    {
        return 'default'; // Override in model for custom log groups
    }

    protected function getDescriptionForEvent(string $event): string
    {
        return class_basename($this) . " was {$event}";
    }

    protected function getPropertiesForEvent(string $event, array $changes): array
    {
        if ($event === 'updated') {
            return [
                'old' => array_intersect_key($this->getOriginal(), $changes),
                'new' => $changes,
            ];
        }

        return []; // For create/delete, no extra properties by default
    }
}
```

Explanation of the trait:
- `bootLogsActivity`: Static boot method that attaches listeners to Eloquent events.
- `logActivity`: Creates the log entry. It uses `Auth::user()` to get the causer (make sure to handle unauthenticated cases if needed).
- Customizable methods like `getDescriptionForEvent` allow per-model overrides.
- For updates, it logs old/new values using `$model->getChanges()` and `$model->getOriginal()`.

### Step 5: Attach the Trait to Your Models
For any model you want to log (e.g., `app/Models/User.php`), add the trait:

```php
<?php

namespace App\Models;

// ... other imports

use App\Traits\LogsActivity;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable, LogsActivity; // Add LogsActivity here

    // Optional: Override methods if needed
    protected function getLogName(): string
    {
        return 'user_logs'; // Custom log name for this model
    }

    // ... rest of your model
}
```

Now, whenever a `User` is created, updated, or deleted, a log entry will be created automatically.

### Step 6: Testing and Usage
- **Create a record**: E.g., `User::create([...])` → Logs "User was created".
- **Update a record**: E.g., `$user->update(['name' => 'New Name'])` → Logs "User was updated" with old/new properties.
- **Delete a record**: E.g., `$user->delete()` → Logs "User was deleted".

To view logs:
- Query the `ActivityLog` model, e.g., `ActivityLog::latest()->get()`.
- You can filter by `subject_type`, `causer_id`, etc.

### Step 7: Enhancements and Considerations
- **Soft Deletes**: If your models use soft deletes (`use SoftDeletes;`), add listeners for `trashed`, `restored`, `forceDeleted`.
- **Batch Operations**: This won't log bulk updates/deletes (e.g., `User::where(...)->update([...])`) because Eloquent events don't fire on query builders. You'd need middleware or custom logic for that.
- **Custom Events**: You can manually call `$model->logActivity('custom_event', $data)` from anywhere.
- **Performance**: Logging adds overhead; consider queuing the log creation if it's high-traffic (use Laravel queues).
- **Security**: Ensure only necessary fields are logged (e.g., exclude passwords via model's `$hidden`).
- **Properties**: The `properties` field can store anything as JSON—expand `getPropertiesForEvent` for more details.
- **Unauthenticated Actions**: If no user is logged in, `causer_id` will be null. You could log IP or other info instead.
- **Validation**: Add error handling in `logActivity` if needed (e.g., if Auth::user() is null).

This setup mimics the core of `spatie/laravel-activitylog`. If you need more features (e.g., pruning old logs), you can extend it further. If you run into issues, share specific error details!


### 🔴  please implement this part "Logging adds overhead; consider queuing the log creation if it's high-traffic (use Laravel queues).", make it like developer can set if he want to queue the logging (because in dev mode developer do not need queue for performance but needed in production mode)


### Step 1: Set Up Configuration for Queuing
To make queuing configurable (e.g., disabled in development for immediate feedback, enabled in production for performance), we'll use Laravel's configuration system. Create a new config file: `config/activitylog.php` (if it doesn't exist).

Add the following content to `config/activitylog.php`:

```php
<?php

return [
    'queue' => [
        'enabled' => env('ACTIVITYLOG_QUEUE_ENABLED', false), // Default to false (no queuing)
        'queue_name' => env('ACTIVITYLOG_QUEUE_NAME', 'default'), // Queue name (e.g., 'logs' for a dedicated queue)
    ],
];
```

In your `.env` file, you can toggle it:
- For development: `ACTIVITYLOG_QUEUE_ENABLED=false`
- For production: `ACTIVITYLOG_QUEUE_ENABLED=true`
- Optionally: `ACTIVITYLOG_QUEUE_NAME=logs` (if you want a custom queue; configure it in `config/queue.php` connections)

This allows easy switching without code changes.

### Step 2: Create a Queue Job for Log Creation
Run the following Artisan command to create a job:

```
php artisan make:job CreateActivityLog
```

Edit `app/Jobs/CreateActivityLog.php`:

```php
<?php

namespace App\Jobs;

use App\Models\ActivityLog;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class CreateActivityLog implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected array $logData;

    public function __construct(array $logData)
    {
        $this->logData = $logData;
    }

    public function handle(): void
    {
        ActivityLog::create($this->logData);
    }
}
```

This job takes the log data as a constructor argument and creates the `ActivityLog` entry when processed by the queue worker.

### Step 3: Update the LogsActivity Trait
Modify `app/Traits/LogsActivity.php` to check the config and either create the log synchronously or dispatch the job.

Update the `logActivity` method:

```php
<?php

namespace App\Traits;

use App\Jobs\CreateActivityLog;
use App\Models\ActivityLog;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Auth;

trait LogsActivity
{
    // ... (keep the bootLogsActivity and other methods as before)

    protected function logActivity(string $event, array $changes = []): void
    {
        $description = $this->getDescriptionForEvent($event);

        $log = [
            'log_name' => $this->getLogName(),
            'description' => $description,
            'subject_id' => $this->getKey(),
            'subject_type' => get_class($this),
            'causer_id' => Auth::id(),
            'causer_type' => Auth::user() ? get_class(Auth::user()) : null,
            'properties' => $this->getPropertiesForEvent($event, $changes),
        ];

        if (config('activitylog.queue.enabled', false)) {
            // Queue the log creation
            CreateActivityLog::dispatch($log)
                ->onQueue(config('activitylog.queue.queue_name', 'default'));
        } else {
            // Create synchronously
            ActivityLog::create($log);
        }
    }

    // ... (rest of the trait remains the same)
}
```

### Step 4: Configure Queues in Laravel
Ensure your queue system is set up:
- In `config/queue.php`, choose a driver (e.g., `database`, `redis`). For example, if using database:
  - Run `php artisan queue:table` and `php artisan migrate` to create the jobs table.
- Start a queue worker: `php artisan queue:work` (or use Supervisor for production).
- If using a custom queue name like 'logs', add it to your queue worker command: `php artisan queue:work --queue=default,logs`.

### Step 5: Usage and Testing
- In dev mode (`.env`: `ACTIVITYLOG_QUEUE_ENABLED=false`), logs are created immediately (synchronous, no queue overhead).
- In production (`.env`: `ACTIVITYLOG_QUEUE_ENABLED=true`), logs are queued, reducing request time but delaying log availability slightly.
- Test by triggering events (create/update/delete) and checking the `activity_logs` table or failed jobs if any.
- If needed, override per-model: Add a protected property `$shouldQueueLogs` in your model and check it instead of/in addition to config.

This keeps the system flexible—toggle via env without redeploying code. If you have multiple environments, use `.env.local` or deployment tools to manage the setting. Let me know if you need more tweaks!
