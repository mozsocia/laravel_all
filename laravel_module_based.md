

# Laravel Modular Architecture — Complete Guide (Zero Packages)

---

## Phase 1: Project Configuration

### 1.1 `composer.json`

```json
"autoload": {
    "psr-4": {
        "App\\": "app/",
        "Modules\\": "Modules/",
        "Database\\Factories\\": "database/factories/",
        "Database\\Seeders\\": "database/seeders/"
    }
}
```

```bash
composer dump-autoload
```

### 1.2 `phpunit.xml` — add the Modules test suite

```xml
<testsuites>
    <testsuite name="Unit">
        <directory>tests/Unit</directory>
    </testsuite>
    <testsuite name="Feature">
        <directory>tests/Feature</directory>
    </testsuite>
    <!-- Add this -->
    <testsuite name="Modules">
        <directory>Modules/*/Tests</directory>
    </testsuite>
</testsuites>
```

### 1.3 `bootstrap/providers.php` — one-time registration

```php
<?php

return [
    App\Providers\AppServiceProvider::class,
    App\Providers\ModuleServiceProvider::class, // ← Only this. Never touch again.
];
```

---

## Phase 2: Core Infrastructure (Write Once)

### 2.1 `app/Providers/ModuleServiceProvider.php`

Auto-discovers every module. Adding a module = creating a folder.

```php
<?php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\File;

class ModuleServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $modulesPath = base_path('Modules');

        if (! is_dir($modulesPath)) {
            return;
        }

        foreach (File::directories($modulesPath) as $modulePath) {
            $this->registerModuleProviders($modulePath);
        }
    }

    protected function registerModuleProviders(string $modulePath): void
    {
        $providersPath = $modulePath . '/Providers';

        if (! is_dir($providersPath)) {
            return;
        }

        collect(File::files($providersPath))
            ->filter(fn ($file) => str_ends_with($file->getFilename(), 'ServiceProvider.php'))
            ->each(function ($file) use ($modulePath) {
                $module = basename($modulePath);
                $class  = "Modules\\{$module}\\Providers\\" . $file->getBasename('.php');

                if (class_exists($class)) {
                    $this->app->register($class);
                }
            });
    }
}
```

---

### 2.2 `app/Providers/BaseModuleServiceProvider.php`

The heart. Loads routes, views, migrations, config, translations, commands, and Blade components — **only if the folder exists**.

```php
<?php

namespace App\Providers;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Blade;
use Illuminate\Support\Facades\File;
use Illuminate\Support\Facades\Route;
use Illuminate\Support\ServiceProvider;

abstract class BaseModuleServiceProvider extends ServiceProvider
{
    /**
     * The PascalCase module name, e.g. 'Accounting'.
     * This single property drives everything.
     */
    protected string $moduleName;

    // ── Derived Helpers ─────────────────────────────────────────

    protected function modulePath(string $sub = ''): string
    {
        return base_path("Modules/{$this->moduleName}" . ($sub ? "/$sub" : ''));
    }

    protected function moduleAlias(): string
    {
        return strtolower($this->moduleName);
    }

    // ── Boot ────────────────────────────────────────────────────

    public function boot(): void
    {
        $this->bootRoutes();
        $this->bootViews();
        $this->bootMigrations();
        $this->bootConfig();
        $this->bootTranslations();
        $this->bootCommands();
        $this->bootComponents();
    }

    // ── Routes ──────────────────────────────────────────────────

    protected function bootRoutes(): void
    {
        $alias = $this->moduleAlias();

        if (file_exists($f = $this->modulePath('Routes/web.php'))) {
            Route::middleware('web')
                ->prefix($alias)
                ->name("{$alias}.")
                ->group($f);
        }

        if (file_exists($f = $this->modulePath('Routes/api.php'))) {
            Route::middleware('api')
                ->prefix("api/{$alias}")
                ->name("api.{$alias}.")
                ->group($f);
        }
    }

    // ── Views ───────────────────────────────────────────────────

    protected function bootViews(): void
    {
        if (is_dir($d = $this->modulePath('Resources/Views'))) {
            $this->loadViewsFrom($d, $this->moduleAlias());
        }
    }

    // ── Migrations ──────────────────────────────────────────────

    protected function bootMigrations(): void
    {
        if (is_dir($d = $this->modulePath('Database/Migrations'))) {
            $this->loadMigrationsFrom($d);
        }
    }

    // ── Config ──────────────────────────────────────────────────

    protected function bootConfig(): void
    {
        $alias = $this->moduleAlias();

        if (file_exists($f = $this->modulePath("Config/{$alias}.php"))) {
            $this->mergeConfigFrom($f, $alias);
            $this->publishes([$f => config_path("{$alias}.php")], "{$alias}-config");
        }
    }

    // ── Translations ────────────────────────────────────────────

    protected function bootTranslations(): void
    {
        if (is_dir($d = $this->modulePath('Lang'))) {
            $this->loadTranslationsFrom($d, $this->moduleAlias());
        }
    }

    // ── Console Commands ────────────────────────────────────────

    protected function bootCommands(): void
    {
        if (! $this->app->runningInConsole()) {
            return;
        }

        if (! is_dir($dir = $this->modulePath('Console/Commands'))) {
            return;
        }

        $namespace = "Modules\\{$this->moduleName}\\Console\\Commands";

        $commands = collect(File::allFiles($dir))
            ->filter(fn ($f) => $f->getExtension() === 'php')
            ->map(fn ($f) => $namespace . '\\' . str_replace(
                ['/', '.php'], ['\\', ''], $f->getRelativePathname()
            ))
            ->filter(fn ($c) => class_exists($c) && is_subclass_of($c, Command::class))
            ->all();

        if ($commands) {
            $this->commands($commands);
        }
    }

    // ── Blade Anonymous Components ──────────────────────────────

    protected function bootComponents(): void
    {
        if (is_dir($d = $this->modulePath('Resources/Views/components'))) {
            Blade::anonymousComponentPath($d, $this->moduleAlias());
        }
    }
}
```

---

### 2.3 `app/Console/Commands/MakeModule.php`

Scaffolds a full module with one command.

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Filesystem\Filesystem;
use Illuminate\Support\Str;

class MakeModule extends Command
{
    protected $signature   = 'make:module {name : PascalCase module name, e.g. Accounting}';
    protected $description = 'Scaffold a new module with all directories and boilerplate';

    protected const DIRECTORIES = [
        'Config',
        'Console/Commands',
        'Contracts',
        'Controllers',
        'Database/Factories',
        'Database/Migrations',
        'Database/Seeders',
        'Lang/en',
        'Middleware',
        'Models',
        'Providers',
        'Requests',
        'Resources/Views/components',
        'Routes',
        'Services',
        'Tests/Feature',
        'Tests/Unit',
    ];

    public function __construct(protected Filesystem $files)
    {
        parent::__construct();
    }

    public function handle(): int
    {
        $name = Str::studly($this->argument('name'));
        $path = base_path("Modules/{$name}");

        if ($this->files->isDirectory($path)) {
            $this->error("Module [{$name}] already exists.");
            return self::FAILURE;
        }

        // Directories
        foreach (self::DIRECTORIES as $dir) {
            $this->files->makeDirectory("{$path}/{$dir}", 0755, true);
        }

        // Boilerplate files
        $alias = strtolower($name);
        $this->files->put("{$path}/Providers/{$name}ServiceProvider.php", $this->providerStub($name));
        $this->files->put("{$path}/Routes/web.php", $this->routeStub('web'));
        $this->files->put("{$path}/Routes/api.php", $this->routeStub('api'));
        $this->files->put("{$path}/Config/{$alias}.php", $this->configStub($name));

        $this->info("Module [{$name}] created at Modules/{$name}");
        $this->line('  Auto-discovered via ModuleServiceProvider — no registration needed.');

        return self::SUCCESS;
    }

    // ── Stubs ───────────────────────────────────────────────────

    protected function providerStub(string $name): string
    {
        return <<<PHP
        <?php

        namespace Modules\\{$name}\\Providers;

        use App\\Providers\\BaseModuleServiceProvider;

        class {$name}ServiceProvider extends BaseModuleServiceProvider
        {
            protected string \$moduleName = '{$name}';
        }
        PHP;
    }

    protected function routeStub(string $type): string
    {
        return <<<PHP
        <?php

        use Illuminate\\Support\\Facades\\Route;

        // Define your {$type} routes here.
        PHP;
    }

    protected function configStub(string $name): string
    {
        return <<<PHP
        <?php

        return [
            'name' => '{$name}',
        ];
        PHP;
    }
}
```

---

### 2.4 `database/seeders/ModuleSeeder.php`

Auto-discovers and runs all module seeders.

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Facades\File;

class ModuleSeeder extends Seeder
{
    public function run(): void
    {
        $modulesPath = base_path('Modules');

        if (! is_dir($modulesPath)) {
            return;
        }

        foreach (File::directories($modulesPath) as $modulePath) {
            $seedersDir = "{$modulePath}/Database/Seeders";

            if (! is_dir($seedersDir)) {
                continue;
            }

            collect(File::files($seedersDir))
                ->filter(fn ($f) => $f->getExtension() === 'php')
                ->each(function ($file) use ($modulePath) {
                    $module = basename($modulePath);
                    $class  = "Modules\\{$module}\\Database\\Seeders\\" . $file->getBasename('.php');

                    if (class_exists($class)) {
                        $this->call($class);
                    }
                });
        }
    }
}
```

Wire it into your main seeder:

```php
// database/seeders/DatabaseSeeder.php
public function run(): void
{
    $this->call(ModuleSeeder::class);
}
```

---

## Phase 3: Creating a Module

```bash
php artisan make:module Accounting
```

This creates:

```
Modules/
└── Accounting/
    ├── Config/
    │   └── accounting.php
    ├── Console/Commands/
    ├── Contracts/
    ├── Controllers/
    ├── Database/
    │   ├── Factories/
    │   ├── Migrations/
    │   └── Seeders/
    ├── Lang/en/
    ├── Middleware/
    ├── Models/
    ├── Providers/
    │   └── AccountingServiceProvider.php
    ├── Requests/
    ├── Resources/Views/components/
    ├── Routes/
    │   ├── api.php
    │   └── web.php
    ├── Services/
    └── Tests/
        ├── Feature/
        └── Unit/
```

**That's it.** No registration anywhere. The `ModuleServiceProvider` discovers it automatically on next request.

---

## Phase 4: Complete Example — Accounting Module

### 4.1 Service Provider

`Modules/Accounting/Providers/AccountingServiceProvider.php`

```php
<?php

namespace Modules\Accounting\Providers;

use App\Providers\BaseModuleServiceProvider;
use Modules\Accounting\Contracts\InvoiceServiceInterface;
use Modules\Accounting\Services\InvoiceService;

class AccountingServiceProvider extends BaseModuleServiceProvider
{
    protected string $moduleName = 'Accounting';

    public function register(): void
    {
        // Bind interfaces for inter-module communication
        $this->app->bind(InvoiceServiceInterface::class, InvoiceService::class);
    }

    // boot() is inherited — loads routes, views, migrations, etc. automatically.
    // Override boot() only if you need custom logic (call parent::boot() first).
}
```

---

### 4.2 Routes

`Modules/Accounting/Routes/web.php`

```php
<?php

// Routes are automatically prefixed with 'accounting'
// and named with 'accounting.' by BaseModuleServiceProvider.
//
// So Route::get('/invoices', ...) becomes:
//   URL:  /accounting/invoices
//   Name: accounting.invoices.index

use Illuminate\Support\Facades\Route;
use Modules\Accounting\Controllers\InvoiceController;

Route::middleware('auth')->group(function () {
    Route::resource('invoices', InvoiceController::class);
});
```

`Modules/Accounting/Routes/api.php`

```php
<?php

// Prefixed with 'api/accounting', named 'api.accounting.'

use Illuminate\Support\Facades\Route;
use Modules\Accounting\Controllers\Api\InvoiceApiController;

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('invoices', InvoiceApiController::class);
});
```

---

### 4.3 Controller

`Modules/Accounting/Controllers/InvoiceController.php`

```php
<?php

namespace Modules\Accounting\Controllers;

use App\Http\Controllers\Controller;
use Modules\Accounting\Contracts\InvoiceServiceInterface;
use Modules\Accounting\Models\Invoice;
use Modules\Accounting\Requests\StoreInvoiceRequest;

class InvoiceController extends Controller
{
    public function __construct(
        protected InvoiceServiceInterface $invoices
    ) {}

    public function index()
    {
        return view('accounting::invoices.index', [
            'invoices' => Invoice::latest()->paginate(15),
        ]);
    }

    public function create()
    {
        return view('accounting::invoices.create');
    }

    public function store(StoreInvoiceRequest $request)
    {
        $invoice = $this->invoices->create($request->validated());

        return redirect()
            ->route('accounting.invoices.show', $invoice)
            ->with('success', 'Invoice created.');
    }

    public function show(Invoice $invoice)
    {
        return view('accounting::invoices.show', compact('invoice'));
    }
}
```

---

### 4.4 Model + Factory + Migration + Seeder

`Modules/Accounting/Models/Invoice.php`

```php
<?php

namespace Modules\Accounting\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Invoice extends Model
{
    use HasFactory;

    protected $fillable = ['amount', 'status', 'due_date'];

    protected $casts = [
        'amount'   => 'decimal:2',
        'due_date' => 'date',
    ];

    // Required: Laravel can't auto-resolve factories outside app/
    protected static function newFactory()
    {
        return \Modules\Accounting\Database\Factories\InvoiceFactory::new();
    }
}
```

`Modules/Accounting/Database/Factories/InvoiceFactory.php`

```php
<?php

namespace Modules\Accounting\Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use Modules\Accounting\Models\Invoice;

class InvoiceFactory extends Factory
{
    protected $model = Invoice::class;

    public function definition(): array
    {
        return [
            'amount'   => fake()->randomFloat(2, 100, 10000),
            'status'   => fake()->randomElement(['draft', 'sent', 'paid']),
            'due_date' => fake()->dateTimeBetween('+1 week', '+3 months'),
        ];
    }
}
```

`Modules/Accounting/Database/Migrations/2025_01_01_000001_create_invoices_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('invoices', function (Blueprint $table) {
            $table->id();
            $table->decimal('amount', 12, 2);
            $table->string('status')->default('draft');
            $table->date('due_date')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('invoices');
    }
};
```

`Modules/Accounting/Database/Seeders/AccountingSeeder.php`

```php
<?php

namespace Modules\Accounting\Database\Seeders;

use Illuminate\Database\Seeder;
use Modules\Accounting\Models\Invoice;

class AccountingSeeder extends Seeder
{
    public function run(): void
    {
        Invoice::factory(25)->create();
    }
}
```

---

### 4.5 Service + Contract (Interface)

`Modules/Accounting/Contracts/InvoiceServiceInterface.php`

```php
<?php

namespace Modules\Accounting\Contracts;

use Modules\Accounting\Models\Invoice;

interface InvoiceServiceInterface
{
    public function create(array $data): Invoice;
}
```

`Modules/Accounting/Services/InvoiceService.php`

```php
<?php

namespace Modules\Accounting\Services;

use Modules\Accounting\Contracts\InvoiceServiceInterface;
use Modules\Accounting\Models\Invoice;

class InvoiceService implements InvoiceServiceInterface
{
    public function create(array $data): Invoice
    {
        return Invoice::create($data);
    }
}
```

---

### 4.6 Form Request

`Modules/Accounting/Requests/StoreInvoiceRequest.php`

```php
<?php

namespace Modules\Accounting\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StoreInvoiceRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'amount'   => ['required', 'numeric', 'min:0.01'],
            'status'   => ['required', 'in:draft,sent,paid'],
            'due_date' => ['required', 'date', 'after:today'],
        ];
    }
}
```

---

### 4.7 Config

`Modules/Accounting/Config/accounting.php`

```php
<?php

return [
    'name'          => 'Accounting',
    'tax_rate'      => env('ACCOUNTING_TAX_RATE', 0.10),
    'currency'      => env('ACCOUNTING_CURRENCY', 'USD'),
    'invoice_prefix' => env('ACCOUNTING_INVOICE_PREFIX', 'INV-'),
];
```

Access anywhere: `config('accounting.tax_rate')`

Publish to override: `php artisan vendor:publish --tag=accounting-config`

---

### 4.8 View

`Modules/Accounting/Resources/Views/invoices/index.blade.php`

```blade
@extends('layouts.app')

@section('content')
    <h1>Invoices</h1>

    @foreach ($invoices as $invoice)
        <div>
            {{ config('accounting.invoice_prefix') }}{{ $invoice->id }}
            — {{ $invoice->amount }} {{ config('accounting.currency') }}
            — {{ $invoice->status }}
        </div>
    @endforeach

    {{ $invoices->links() }}
@endsection
```

Blade anonymous component example — `Modules/Accounting/Resources/Views/components/status-badge.blade.php`:

```blade
@props(['status'])

<span class="badge badge-{{ $status }}">
    {{ ucfirst($status) }}
</span>
```

Use it anywhere: `<x-accounting::status-badge status="paid" />`

---

### 4.9 Test

`Modules/Accounting/Tests/Feature/InvoiceTest.php`

```php
<?php

namespace Modules\Accounting\Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Modules\Accounting\Models\Invoice;
use Tests\TestCase;

class InvoiceTest extends TestCase
{
    use RefreshDatabase;

    public function test_can_list_invoices(): void
    {
        Invoice::factory(3)->create();

        $this->actingAs(User::factory()->create())
            ->get(route('accounting.invoices.index'))
            ->assertOk()
            ->assertViewHas('invoices');
    }

    public function test_can_create_invoice(): void
    {
        $this->actingAs(User::factory()->create())
            ->post(route('accounting.invoices.store'), [
                'amount'   => 500.00,
                'status'   => 'draft',
                'due_date' => now()->addMonth()->toDateString(),
            ])
            ->assertRedirect();

        $this->assertDatabaseHas('invoices', ['amount' => 500.00]);
    }
}
```

---

## Phase 5: Inter-Module Communication

Modules talk through **interfaces**, never concrete classes.

```
Accounting module                  Reporting module
┌──────────────────────┐           ┌──────────────────────┐
│ Contracts/            │           │ Controllers/          │
│   InvoiceServiceIF ◄─┼───────────┼── ReportController    │
│                       │           │     (depends on IF)   │
│ Services/             │           │                       │
│   InvoiceService      │           │                       │
│     (implements IF)   │           │                       │
└──────────────────────┘           └──────────────────────┘
```

**In the consuming module:**

```php
// Modules/Reporting/Controllers/ReportController.php
namespace Modules\Reporting\Controllers;

use Modules\Accounting\Contracts\InvoiceServiceInterface;

class ReportController
{
    public function __construct(
        protected InvoiceServiceInterface $invoices  // Resolved via DI
    ) {}
}
```

The binding was made in `AccountingServiceProvider::register()`. Laravel resolves it automatically. **Zero coupling to the concrete class.**

---

## Phase 6: Quick Reference

| Task | How |
|---|---|
| **Create module** | `php artisan make:module Billing` |
| **Return view** | `return view('accounting::invoices.index');` |
| **Named route** | `route('accounting.invoices.show', $invoice)` |
| **Blade component** | `<x-accounting::status-badge status="paid" />` |
| **Read config** | `config('accounting.tax_rate')` |
| **Translation** | `__('accounting::messages.created')` |
| **Run migrations** | `php artisan migrate` (auto-discovered) |
| **Seed all modules** | `php artisan db:seed` (via `ModuleSeeder`) |
| **Seed one module** | `php artisan db:seed --class=Modules\\Accounting\\Database\\Seeders\\AccountingSeeder` |
| **Run module tests** | `php artisan test --testsuite=Modules` |
| **Publish config** | `php artisan vendor:publish --tag=accounting-config` |

---

## Key Design Decisions

| Concern | Decision |
|---|---|
| **Route collisions** | Every module gets `prefix` + `name` prefix automatically |
| **No `->namespace()`** | Modern Laravel uses `use` imports in route files — no double-namespacing |
| **One property per module** | Child providers set only `$moduleName`; everything else is derived |
| **Convention over configuration** | Folder exists → it's loaded. No folder → silently skipped |
| **Override anything** | Every `boot*()` method is `protected` — override in your provider |
| **Config caching** | Run `vendor:publish` for all module configs before `php artisan config:cache` in production |

**To remove the web route prefix** for a specific module (e.g., a Dashboard module), override `bootRoutes()`:

```php
class DashboardServiceProvider extends BaseModuleServiceProvider
{
    protected string $moduleName = 'Dashboard';

    protected function bootRoutes(): void
    {
        if (file_exists($f = $this->modulePath('Routes/web.php'))) {
            Route::middleware('web')->name('dashboard.')->group($f);
        }
    }
}
```
