### 🔴 i will do agent manual loop and do all agent chaining by my self so i need very simple libraray which will do tool defination and LLM call

Since you want to handle the **agent loop and chaining manually** (like checking for tool calls, executing tools yourself, appending tool results as messages, and repeating calls up to N steps), the **simplest and most straightforward library** in Laravel/PHP is **moe-mizrak/laravel-openrouter**.

### Why this is the best fit for your needs:
- It's specifically built for **OpenRouter** (exactly what you're using in your TS code with `@openrouter/ai-sdk-provider`).
- Very lightweight and simple: just a thin Laravel wrapper around the OpenRouter API (which is 100% OpenAI-compatible).
- Full support for **tool calling** (function calling) – you pass tools as OpenAI-style JSON schemas.
- No automatic orchestration or agent loops – perfect, since you want to control the multi-step chaining yourself.
- Easy facade usage, DTO responses, streaming support if needed later.
- Much simpler than openai-php/laravel (which requires manual base URL override for OpenRouter) or Prism (which adds provider abstraction and optional auto-tool handling you don't want).

### Alternative if you prefer the most popular base:
Use **openai-php/laravel** (the most widely used, ~5.6k GitHub stars on the client) and configure it to point to OpenRouter's endpoint. It's nearly identical in usage, just one extra config step.

But since you're already on OpenRouter, **laravel-openrouter** is cleaner and more direct.

Let me know which one you prefer ("Go with laravel-openrouter" or "Use openai-php/laravel"), and I'll provide:
- Full installation & config steps
- How to define tools (similar to your Zod + tool pattern, using JSON schemas)
- A complete route/controller example mirroring your `/agentv2/run` endpoint
- Manual loop implementation with step counting, tool execution, and response collection (like `toolResults`, `steps`, `text`)

Just say the word!


### 🔴 if i use openai-php/laravel can i switch any open ai compatible provider easily? because sometimes i want to use deepseek ai, minimax ai , open ai,


Yes, **you can easily switch between any OpenAI-compatible providers** (like **DeepSeek**, **Minimax**, **Groq**, **OpenRouter**, or even local servers) using **openai-php/laravel**.

### Why It Works Seamlessly
The package wraps the official `openai-php/client` library, which supports a **custom base URL** out of the box. All the providers you mentioned (DeepSeek, Minimax, etc.) offer **fully OpenAI-compatible APIs**, so you just change the endpoint (base URL) and API key — the rest of the code (chat completions, tool calling, streaming, etc.) stays exactly the same.

### How to Switch Providers Easily
You have a few flexible options, from simplest to most dynamic:

1. **Quickest way (per request or in code)**  
   Create a new client instance on the fly with a different base URL and key:

   ```php
   use OpenAI;

   // For official OpenAI
   $openaiClient = OpenAI::client(env('OPENAI_API_KEY'));

   // For DeepSeek
   $deepseekClient = OpenAI::factory()
       ->withApiKey(env('DEEPSEEK_API_KEY'))
       ->withBaseUri('https://api.deepseek.com/v1') // or just https://api.deepseek.com
       ->make();

   // For Groq (very fast inference)
   $groqClient = OpenAI::factory()
       ->withApiKey(env('GROQ_API_KEY'))
       ->withBaseUri('https://api.groq.com/openai/v1')
       ->make();

   // For Minimax (if they provide OpenAI-compatible endpoint)
   $minimaxClient = OpenAI::factory()
       ->withApiKey(env('MINIMAX_API_KEY'))
       ->withBaseUri('https://api.minimax.com/v1') // check their docs for exact path
       ->make();

   // Then use exactly the same methods
   $response = $deepseekClient->chat()->create([...]);
   ```

   This is perfect for your manual agent loop — you can decide per request (or per user/feature) which provider to use.

2. **Config-based (multiple named clients)**  
   Publish the config (`php artisan vendor:publish --tag=openai-config`) and extend it to support multiple providers, or just override the default in `.env` when you want to switch globally:

   ```env
   # Default (OpenAI)
   OPENAI_API_KEY=sk-...

   # Switch to DeepSeek globally
   # OPENAI_API_KEY=your-deepseek-key
   # OPENAI_BASE_URL=https://api.deepseek.com/v1
   ```

   Some forks/community versions already support `OPENAI_BASE_URL` in config, but with the factory method above it's built-in anyway.

3. **Even easier long-term switching**  
   If you want a clean abstraction later (e.g., fallback or routing logic), you can wrap the client in a simple service class or use a multi-provider package like Prism (which uses openai-php under the hood for OpenAI-compatible ones). But since you want manual control and simplicity, the factory method is ideal.

### Summary
- **Yes, switching is very easy** — just change the base URI and API key.
- No code changes needed for the actual LLM calls or tool definitions.
- Works perfectly with your manual tool loop and chaining.
- DeepSeek, Groq, OpenRouter, and most modern providers are 100% compatible this way.

If you go with **openai-php/laravel**, you'll have the most popular, well-maintained base with full flexibility for exactly the providers you mentioned.

Ready for the full code example (installation + manual agent loop with tool calling, mirroring your TS version)? Just say "Go with openai-php/laravel" and I'll deliver it!


### 🔴 Go with openai-php/laravel , Full code example with tool calling

Here is a **complete, ready-to-use Laravel implementation** using **openai-php/laravel** that mirrors your TypeScript example as closely as possible.

You will:
- Define tools with description and JSON schema (similar to Zod)
- Manually handle the agent loop (up to 5 steps, like `stopWhen: stepCountIs(5)`)
- Collect `text`, `toolResults`, `steps`, and `usage`
- Support easy switching between OpenRouter, DeepSeek, OpenAI, etc.

### 1. Installation

```bash
composer require openai-php/laravel
php artisan vendor:publish --provider="OpenAI\Laravel\ServiceProvider"
```

### 2. Environment (.env)

```env
# Default: OpenRouter (your current provider)
OPENAI_API_KEY=sk-or-...your-openrouter-key
OPENAI_BASE_URL=https://openrouter.ai/api/v1

# Example: Switch to DeepSeek
# OPENAI_API_KEY=your-deepseek-key
# OPENAI_BASE_URL=https://api.deepseek.com/v1

# Example: Switch to official OpenAI
# OPENAI_API_KEY=sk-...
# OPENAI_BASE_URL=https://api.openai.com/v1
```

> Just change these two lines to switch providers instantly — no code changes needed!

### 3. Define Your Tool (similar to your `weatherTool`)

Create a file: `app/Tools/WeatherTool.php`

```php
<?php

namespace App\Tools;

class WeatherTool
{
    public function definition(): array
    {
        return [
            'type' => 'function',
            'function' => [
                'name' => 'get_weather',
                'description' => 'Get current weather for a location',
                'parameters' => [
                    'type' => 'object',
                    'properties' => [
                        'location' => [
                            'type' => 'string',
                            'description' => 'City and country, e.g. Berlin, Germany',
                        ],
                    ],
                    'required' => ['location'],
                    'additionalProperties' => false,
                ],
            ],
        ];
    }

    public function execute(array $arguments): array
    {
        $location = $arguments['location'] ?? 'Unknown';

        // Replace with real API call later
        return [
            'temperature' => 18,
            'condition' => 'Cloudy',
            'location' => $location,
        ];
    }
}
```

### 4. Create the Agent Controller / Route

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;
use OpenAI\Laravel\Facades\OpenAI;
use App\Tools\WeatherTool;

class AgentController extends Controller
{
    public function run(Request $request)
    {
        // Validation similar to your Zod schema
        $validator = Validator::make($request->all(), [
            'userMessage' => 'required|string|min:1|max:10000',
            'model' => 'sometimes|string', // optional: override model
            'maxSteps' => 'sometimes|integer|min:1|max:10',
        ]);

        if ($validator->fails()) {
            return response()->json(['error' => $validator->errors()], 422);
        }

        $userMessage = $request->input('userMessage');
        $modelName = $request->input('model', 'nvidia/nemotron-3-nano-30b-a3b:free'); // default OpenRouter free model
        $maxSteps = $request->input('maxSteps', 5);

        $weatherTool = new WeatherTool();

        $messages = [
            [
                'role' => 'system',
                'content' => 'You are a helpful AI assistant that provides weather information using the get_weather tool when requested.',
            ],
            [
                'role' => 'user',
                'content' => $userMessage,
            ],
        ];

        $tools = [$weatherTool->definition()];

        $steps = [];
        $toolResults = [];
        $finalText = '';
        $usage = ['input_tokens' => 0, 'output_tokens' => 0];

        for ($step = 1; $step <= $maxSteps; $step++) {
            $steps[] = ['step' => $step, 'type' => 'llm_call'];

            $response = OpenAI::chat()->create([
                'model' => $modelName,
                'messages' => $messages,
                'tools' => $tools,
                'tool_choice' => 'auto',
            ]);

            $choice = $response->choices[0];
            $message = $choice->message;

            // Add usage
            $usage['input_tokens'] += $response->usage->promptTokens ?? 0;
            $usage['output_tokens'] += $response->usage->completionTokens ?? 0;

            // Append assistant message
            $messages[] = [
                'role' => $message->role,
                'content' => $message->content,
                'tool_calls' => $message->toolCalls ?? null,
            ];

            if ($message->content) {
                $finalText = $message->content; // last non-null text
            }

            // Check if there are tool calls
            if (!empty($message->toolCalls)) {
                foreach ($message->toolCalls as $toolCall) {
                    $functionName = $toolCall->function->name;
                    $functionArgs = json_decode($toolCall->function->arguments, true);

                    $steps[] = [
                        'step' => $step,
                        'type' => 'tool_call',
                        'name' => $functionName,
                        'arguments' => $functionArgs,
                    ];

                    if ($functionName === 'get_weather') {
                        $result = $weatherTool->execute($functionArgs);

                        $toolResults[] = [
                            'tool' => 'get_weather',
                            'arguments' => $functionArgs,
                            'result' => $result,
                        ];

                        $messages[] = [
                            'role' => 'tool',
                            'tool_call_id' => $toolCall->id,
                            'name' => $functionName,
                            'content' => json_encode($result),
                        ];
                    }
                }

                // Continue loop for next LLM call with tool results
                continue;
            }

            // No tool calls → assistant gave final answer
            break;
        }

        return response()->json([
            'response' => $finalText,
            'steps' => $steps,
            'toolResults' => $toolResults,
            'usage' => $usage,
        ]);
    }
}
```

### 5. Route (routes/api.php or web.php)

```php
use App\Http\Controllers\AgentController;

Route::post('/agent/run', [AgentController::class, 'run']);
```

### Test It

```bash
curl -X POST http://your-app.test/agent/run \
  -H "Content-Type: application/json" \
  -d '{"userMessage": "What is the weather in Paris, France?"}'
```

Expected response:
```json
{
  "response": "The weather in Paris, France is currently cloudy with a temperature of 18°C.",
  "steps": [ ... ],
  "toolResults": [
    {
      "tool": "get_weather",
      "arguments": { "location": "Paris, France" },
      "result": { "temperature": 18, "condition": "Cloudy", "location": "Paris, France" }
    }
  ],
  "usage": { ... }
}
```

You now have full manual control over the agent loop, easy provider switching, and clean tool definitions — all with the most popular and flexible PHP OpenAI library.

Let me know if you want to add multiple tools, streaming, or model routing logic!


### 🔴 Add multiple tool examples


Here’s your **updated and extended version** with **multiple tools** (3 realistic examples), still using **openai-php/laravel** with full manual agent loop control.

### 1. Add More Tool Classes

Create these files in `app/Tools/`:

#### `app/Tools/WeatherTool.php` (already had this)
```php
<?php

namespace App\Tools;

class WeatherTool
{
    public function definition(): array
    {
        return [
            'type' => 'function',
            'function' => [
                'name' => 'get_weather',
                'description' => 'Get the current weather for a given location',
                'parameters' => [
                    'type' => 'object',
                    'properties' => [
                        'location' => [
                            'type' => 'string',
                            'description' => 'City and country, e.g. Berlin, Germany',
                        ],
                    ],
                    'required' => ['location'],
                    'additionalProperties' => false,
                ],
            ],
        ];
    }

    public function execute(array $arguments): array
    {
        $location = $arguments['location'] ?? 'Unknown';

        // Mock data – replace with real API later
        return [
            'location' => $location,
            'temperature_celsius' => 18,
            'condition' => 'Cloudy',
            'humidity' => 65,
        ];
    }
}
```

#### `app/Tools/GetCurrentTimeTool.php`
```php
<?php

namespace App\Tools;

use DateTime;
use DateTimeZone;

class GetCurrentTimeTool
{
    public function definition(): array
    {
        return [
            'type' => 'function',
            'function' => [
                'name' => 'get_current_time',
                'description' => 'Get the current time in a specific timezone',
                'parameters' => [
                    'type' => 'object',
                    'properties' => [
                        'timezone' => [
                            'type' => 'string',
                            'description' => 'IANA timezone name, e.g. Europe/Berlin, America/New_York',
                        ],
                    ],
                    'required' => ['timezone'],
                    'additionalProperties' => false,
                ],
            ],
        ];
    }

    public function execute(array $arguments): array
    {
        $timezoneName = $arguments['timezone'] ?? 'UTC';

        try {
            $tz = new DateTimeZone($timezoneName);
            $now = new DateTime('now', $tz);
            return [
                'timezone' => $timezoneName,
                'current_time' => $now->format('Y-m-d H:i:s'),
                'formatted' => $now->format('l, F j, Y g:i A'),
            ];
        } catch (\Exception $e) {
            return [
                'error' => 'Invalid timezone',
                'timezone_provided' => $timezoneName,
            ];
        }
    }
}
```

#### `app/Tools/CalculatorTool.php`
```php
<?php

namespace App\Tools;

use Exception;

class CalculatorTool
{
    public function definition(): array
    {
        return [
            'type' => 'function',
            'function' => [
                'name' => 'calculate',
                'description' => 'Perform basic arithmetic: add, subtract, multiply, divide',
                'parameters' => [
                    'type' => 'object',
                    'properties' => [
                        'expression' => [
                            'type' => 'string',
                            'description' => 'Math expression like "2 + 3 * 4" or "10 / 2"',
                        ],
                    ],
                    'required' => ['expression'],
                    'additionalProperties' => false,
                ],
            ],
        ];
    }

    public function execute(array $arguments): array
    {
        $expression = trim($arguments['expression'] ?? '');

        // Basic security: only allow numbers and operators
        if (!preg_match('/^[\d+\-*\\/().\s]+$/', $expression)) {
            return ['error' => 'Invalid characters in expression'];
        }

        try {
            // Use eval safely (only math)
            @eval('$result = ' . $expression . ';');

            if (!isset($result)) {
                return ['error' => 'Invalid expression'];
            }

            return [
                'expression' => $expression,
                'result' => $result,
            ];
        } catch (Exception $e) {
            return ['error' => 'Calculation failed', 'message' => $e->getMessage()];
        }
    }
}
```

### 2. Updated AgentController with Multiple Tools

Replace your `AgentController@run` method with this enhanced version:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Validator;
use OpenAI\Laravel\Facades\OpenAI;
use App\Tools\WeatherTool;
use App\Tools\GetCurrentTimeTool;
use App\Tools\CalculatorTool;

class AgentController extends Controller
{
    public function run(Request $request)
    {
        $validator = Validator::make($request->all(), [
            'userMessage' => 'required|string|min:1|max:10000',
            'model' => 'sometimes|string',
            'maxSteps' => 'sometimes|integer|min:1|max:20',
        ]);

        if ($validator->fails()) {
            return response()->json(['error' => $validator->errors()], 422);
        }

        $userMessage = $request->input('userMessage');
        $modelName = $request->input('model', 'nvidia/nemotron-3-nano-30b-a3b:free');
        $maxSteps = $request->input('maxSteps', 5);

        // Instantiate all tools
        $tools = [
            new WeatherTool(),
            new GetCurrentTimeTool(),
            new CalculatorTool(),
        ];

        // Build tool definitions for the API
        $toolDefinitions = array_map(fn($tool) => $tool->definition(), $tools);

        // Map tool name → tool instance for execution
        $toolMap = collect($tools)->mapWithKeys(fn($tool) => [
            $tool->definition()['function']['name'] => $tool
        ])->toArray();

        $messages = [
            [
                'role' => 'system',
                'content' => 'You are a helpful AI assistant. Use the available tools when needed: get_weather, get_current_time, and calculate. Only use tools if explicitly required by the user query.',
            ],
            [
                'role' => 'user',
                'content' => $userMessage,
            ],
        ];

        $steps = [];
        $toolResults = [];
        $finalText = '';
        $usage = ['input_tokens' => 0, 'output_tokens' => 0];

        for ($step = 1; $step <= $maxSteps; $step++) {
            $steps[] = ['step' => $step, 'type' => 'llm_call'];

            $response = OpenAI::chat()->create([
                'model' => $modelName,
                'messages' => $messages,
                'tools' => $toolDefinitions,
                'tool_choice' => 'auto',
            ]);

            $choice = $response->choices[0];
            $message = $choice->message;

            // Accumulate usage
            $usage['input_tokens'] += $response->usage->promptTokens ?? 0;
            $usage['output_tokens'] += $response->usage->completionTokens ?? 0;

            // Append assistant message
            $messages[] = [
                'role' => $message->role,
                'content' => $message->content,
                'tool_calls' => $message->toolCalls ?? null,
            ];

            if ($message->content) {
                $finalText = $message->content;
            }

            // Handle tool calls
            if (!empty($message->toolCalls)) {
                foreach ($message->toolCalls as $toolCall) {
                    $functionName = $toolCall->function->name;
                    $functionArgs = json_decode($toolCall->function->arguments, true);

                    $steps[] = [
                        'step' => $step,
                        'type' => 'tool_call',
                        'name' => $functionName,
                        'arguments' => $functionArgs,
                    ];

                    if (!isset($toolMap[$functionName])) {
                        $result = ['error' => 'Unknown tool'];
                    } else {
                        $result = $toolMap[$functionName]->execute($functionArgs);
                    }

                    $toolResults[] = [
                        'tool' => $functionName,
                        'arguments' => $functionArgs,
                        'result' => $result,
                    ];

                    // Append tool response message
                    $messages[] = [
                        'role' => 'tool',
                        'tool_call_id' => $toolCall->id,
                        'name' => $functionName,
                        'content' => json_encode($result),
                    ];
                }

                // Continue to next step
                continue;
            }

            // No tool calls → final answer
            break;
        }

        return response()->json([
            'response' => $finalText,
            'steps' => $steps,
            'toolResults' => $toolResults,
            'usage' => $usage,
        ]);
    }
}
```

### Test Examples

1. **Weather + Time**
   ```json
   {"userMessage": "What's the weather like in Tokyo right now, and what time is it there?"}
   ```

2. **Math**
   ```json
   {"userMessage": "Can you calculate (15 + 27) * 4 - 10?"}
   ```

3. **Mixed**
   ```json
   {"userMessage": "Tell me the current time in New York, then calculate how many hours ahead Berlin is."}
   ```

You now have:
- Clean, reusable tool classes
- Easy to add more tools (just create class + add to `$tools` array)
- Full manual control over chaining and steps
- Provider switching via `.env` only
- Exact mirror of your original Vercel AI SDK behavior

Let me know if you want to add validation (like Zod), parallel tool calls, or structured output parsing!







