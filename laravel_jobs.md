### 🔴 In development is laravel queue jobs run when i am testing using 'php artisan serve'

No, Laravel queue jobs do not run automatically when using `php artisan serve` in a development environment, unless your queue connection is configured to use the "sync" driver.

### Key Details
- **`php artisan serve`** starts a simple built-in web server for handling HTTP requests during development. It does not include or start a queue worker process, so it won't process queued jobs in the background.
- Laravel's queue system defers tasks (like jobs for emails, notifications, or heavy processing) to a backend like a database or Redis. By default, new Laravel applications use the `database` driver as the queue connection (configured via the `QUEUE_CONNECTION` environment variable in your `.env` file or in `config/queue.php`).
- If your queue driver is set to an asynchronous option like `database`, `redis`, or `sqs`:
  - Jobs will be dispatched and added to the queue during a request, but they won't execute until you manually start a separate queue worker process.
  - To process them, run `php artisan queue:work` (or `php artisan queue:listen`) in a separate terminal window. This worker polls the queue and runs the jobs independently of the web server.
- If your queue driver is set to `sync` (common in development for simplicity):
  - Jobs run immediately and synchronously within the same HTTP request process handled by `php artisan serve`. No separate worker is needed, as there's no actual queuing—it's essentially inline execution for testing purposes.
  - You can switch to this by setting `QUEUE_CONNECTION=sync` in your `.env` file.

### Recommendations for Development
- For quick testing without a worker, use the `sync` driver. This avoids needing multiple terminals but isn't ideal for simulating production behavior (where async queues are preferred to avoid blocking requests).
- If you need to test async queuing, keep the default `database` driver (after setting up the `jobs` table with `php artisan queue:table` and `php artisan migrate`), dispatch your jobs, and run the worker separately.
- In production, use a process manager like Supervisor to keep workers running persistently, as closing the terminal would stop them otherwise.

For more details, check the official Laravel documentation on queues.