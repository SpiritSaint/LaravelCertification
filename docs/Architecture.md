# Architecture

## Request Lifecycle

> Reference: [Documentation 6.x](https://laravel.com/docs/6.x/lifecycle)

The file named **index.php** on **/public** folder contains the following structure:

1. Define **LARAVEL_START** variable setting it with **microtime(TRUE)** function. It probably used to do benchmarking or request timing.
2. Requires **Composer autoload** script located on **/vendor** folder. It makes sense because code using PSR-4 (Class autoload) it really necessary to make this framework works.
3. Requires **Laravel Bootstrap** script located on **/bootstrap** folder and it returns the **app** instance. That bootstrap script initialize the **app** variable and assigns three abstracts contracts using **/app** folder **singleton** classes. Initially **Kernels for Http and Console** and finally **Exception Handler**.  
4. Initialize a **kernel** variable making an **Http Contract** instance using **app** instance. Next, the **Request** is captured and handled by **kernel** instance creating **request** variable. That operation creates the **response** variable. 
5. Response is sent to the client using **send** method of **response** variable.
6. Finally, the **kernel** instance terminates the process using both variables: **request** and **response**.

For more explanations the code is the next:

```php
/**
 * 1. Define LARAVEL_START variable using microtime function. 
 */
define('LARAVEL_START', microtime(true));
```

```php
/**
 * 2. Requires Composer Autoload script on index.php
 */
require __DIR__.'/../vendor/autoload.php';

// ⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️
// Contents of autoload.php
// ⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️⚠️

// Initialize the app variable using Foundation Application class
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);

// Assign app folder HTTP kernel class to contract
$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

// Assign app folder Console Kernel class to contract
$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

// Assign app folder Exception Handler to contract
$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);

// The app variable is returned to index.php
return $app;
```

```php
/**
 * 3. Requires Composer Autoload script
 */
require __DIR__.'/../vendor/autoload.php';
```

```php
/**
 * 4. Initialize kernel variable using recently assigned class of contract
 */
$kernel = $app->make(Illuminate\Contracts\Http\Kernel::class);

// The response variable is created by using handle method of kernel after of request captured
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
```

```php
/**
 * 5. Response is sent to the client
 */
$response->send();
```

```php
/**
 * 6. Kernel terminates the process using request and response variables
 */
$kernel->terminate($request, $response);
```



