Creating a custom Artisan command in Laravel to generate a Blade template from a `.stub` file involves a few steps. Here’s a step-by-step guide:

### Step 1: Create the Custom Command

1. **Generate the command file:**
   Run the following Artisan command to create a new custom command file.
   ```sh
   php artisan make:command GenerateBladeTemplate
   ```

2. **Modify the command file:**
   Open the newly created command file located in `app/Console/Commands/GenerateBladeTemplate.php`.

   ```php
   <?php

   namespace App\Console\Commands;

   use Illuminate\Console\Command;
   use Illuminate\Support\Facades\File;

   class GenerateBladeTemplate extends Command
   {
       /**
        * The name and signature of the console command.
        *
        * @var string
        */
       protected $signature = 'make:blade {name}';

       /**
        * The console command description.
        *
        * @var string
        */
       protected $description = 'Generate a new Blade template from a stub file';

       /**
        * Execute the console command.
        *
        * @return int
        */
       public function handle()
       {
           // Get the name argument
           $name = $this->argument('name');

           // Define the paths
           $stubPath = resource_path('stubs/template.stub');
           $bladePath = resource_path("views/{$name}.blade.php");

           // Check if the stub file exists
           if (!File::exists($stubPath)) {
               $this->error('Stub file does not exist.');
               return 1;
           }

           // Get the contents of the stub file
           $stubContent = File::get($stubPath);

           // Create the Blade file with the stub content
           File::put($bladePath, $stubContent);

           $this->info("Blade template created at resources/views/{$name}.blade.php");

           return 0;
       }
   }
   ```

### Step 2: Create the Stub File

1. **Create a directory for stubs:**
   In your `resources` directory, create a `stubs` directory if it doesn’t already exist.

2. **Create the stub file:**
   Inside the `resources/stubs` directory, create a file named `template.stub`.

   ```blade
   {{-- This is a generated Blade template --}}

   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>{{ $title ?? 'Default Title' }}</title>
   </head>
   <body>
       <h1>{{ $heading ?? 'Hello, World!' }}</h1>
       <p>{{ $content ?? 'Welcome to your new Blade template.' }}</p>
   </body>
   </html>
   ```

### Step 3: Test the Command

1. **Run the custom command:**
   Use the Artisan CLI to run your new command and generate a Blade template.

   ```sh
   php artisan make:blade example
   ```

   This will create a file named `example.blade.php` in the `resources/views` directory with the contents from the `template.stub` file.

### Step 4: Customize and Extend (Optional)

You can customize the stub file and the command as needed. For example, you can add more dynamic replacements in the stub file and handle them in your command.

To replace placeholders in your stub file dynamically, modify the `handle` method to replace these placeholders:

```php
// Replace placeholders in the stub content
$stubContent = str_replace(
    ['{{ title }}', '{{ heading }}', '{{ content }}'],
    ['My Title', 'My Heading', 'My Content'],
    $stubContent
);
```

Update your stub file accordingly to use placeholders like `{{ title }}`, `{{ heading }}`, and `{{ content }}`.

With these steps, you've created a Laravel custom command to generate a Blade template from a `.stub` file.



Sure! Let's extend the custom command to allow dynamic replacements in the stub file. We'll modify the command to accept additional options for `title`, `heading`, and `content`, and then replace placeholders in the stub file with these values.

### Step-by-Step Customization and Extension

#### Step 1: Modify the Command

1. **Update the command signature to accept options:**

   ```php
   protected $signature = 'make:blade {name} {--title=} {--heading=} {--content=}';
   ```

2. **Modify the `handle` method to handle dynamic replacements:**

   ```php
   public function handle()
   {
       // Get the arguments and options
       $name = $this->argument('name');
       $title = $this->option('title') ?? 'Default Title';
       $heading = $this->option('heading') ?? 'Hello, World!';
       $content = $this->option('content') ?? 'Welcome to your new Blade template.';

       // Define the paths
       $stubPath = resource_path('stubs/template.stub');
       $bladePath = resource_path("views/{$name}.blade.php");

       // Check if the stub file exists
       if (!File::exists($stubPath)) {
           $this->error('Stub file does not exist.');
           return 1;
       }

       // Get the contents of the stub file
       $stubContent = File::get($stubPath);

       // Replace placeholders in the stub content
       $stubContent = str_replace(
           ['{{ title }}', '{{ heading }}', '{{ content }}'],
           [$title, $heading, $content],
           $stubContent
       );

       // Create the Blade file with the replaced content
       File::put($bladePath, $stubContent);

       $this->info("Blade template created at resources/views/{$name}.blade.php");

       return 0;
   }
   ```

#### Step 2: Update the Stub File

Modify the `template.stub` file to include placeholders for `title`, `heading`, and `content`.

```blade
{{-- This is a generated Blade template --}}

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ $title ?? '{{ title }}' }}</title>
</head>
<body>
    <h1>{{ $heading ?? '{{ heading }}' }}</h1>
    <p>{{ $content ?? '{{ content }}' }}</p>
</body>
</html>
```

#### Step 3: Test the Extended Command

1. **Run the custom command with options:**

   ```sh
   php artisan make:blade example --title="My Custom Title" --heading="My Custom Heading" --content="This is custom content."
   ```

   This command will create a file named `example.blade.php` in the `resources/views` directory with the specified dynamic values.

2. **Check the generated Blade file:**

   The contents of `example.blade.php` should look like this:

   ```blade
   {{-- This is a generated Blade template --}}

   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>My Custom Title</title>
   </head>
   <body>
       <h1>My Custom Heading</h1>
       <p>This is custom content.</p>
   </body>
   </html>
   ```

### Full Code for the Command File

Here is the complete code for `GenerateBladeTemplate.php` with dynamic replacements:

```php
<?php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\File;

class GenerateBladeTemplate extends Command
{
    /**
     * The name and signature of the console command.
     *
     * @var string
     */
    protected $signature = 'make:blade {name} {--title=} {--heading=} {--content=}';

    /**
     * The console command description.
     *
     * @var string
     */
    protected $description = 'Generate a new Blade template from a stub file';

    /**
     * Execute the console command.
     *
     * @return int
     */
    public function handle()
    {
        // Get the arguments and options
        $name = $this->argument('name');
        $title = $this->option('title') ?? 'Default Title';
        $heading = $this->option('heading') ?? 'Hello, World!';
        $content = $this->option('content') ?? 'Welcome to your new Blade template.';

        // Define the paths
        $stubPath = resource_path('stubs/template.stub');
        $bladePath = resource_path("views/{$name}.blade.php");

        // Check if the stub file exists
        if (!File::exists($stubPath)) {
            $this->error('Stub file does not exist.');
            return 1;
        }

        // Get the contents of the stub file
        $stubContent = File::get($stubPath);

        // Replace placeholders in the stub content
        $stubContent = str_replace(
            ['{{ title }}', '{{ heading }}', '{{ content }}'],
            [$title, $heading, $content],
            $stubContent
        );

        // Create the Blade file with the replaced content
        File::put($bladePath, $stubContent);

        $this->info("Blade template created at resources/views/{$name}.blade.php");

        return 0;
    }
}
```

With these modifications, you can now generate Blade templates with dynamic content using your custom Artisan command.
