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

## Service Container Binding and Resolution

As the step 2 of Request Lifecycle, Service Container Binding works in a similar way but  the documentation explains some specific cases:

### When closure is used

Check the following code:

```php
$app->singleton('APIController', function() {
    return new \MyPackage\API\RESTFullController();
});
```

So you can use the next code:

```php
$RESTFullController = $app->instance('APIController');
```

That's simple. 

### When contextual cases are used

Check the following code:

```php
$app->when('APIController')
    ->needs(\App\User::class)
    ->give(function() {
    return \App\User::query();    
});
```

So you can use the next code:

```php
class ApiController {
    public function index(\App\User $users) { 
        return response()->json($users->paginate(25));
    }
}
```

So you don't need to use **query** method. Because $users variable will be a **QueryBuilder**.

### When automatic injection is used

Check the following code:

```php
use CarRepository;

class CarControllers extends \App\Http\Controllers\Controller {
    public function index(CarRepository $carRepository) { 
        return response()->json($carRepository->latestWithUser(100));
    }
}
```

That code will works because **CarRepository** will be injected automatically. I'll use the other implementation to explain this feature more easily:

```php
use CarRepository;

class CarControllers extends \App\Http\Controllers\Controller {
    protected $carRepository;
    public function __construct() {
        $this->carRepository = new CarRepository();
    }
    public function index() { 
        return response()->json($this->carRepository->latestWithUser(100));
    }
}
```

As you can see, **CarRepository** should be initialized using **new** keyword.

### When extending binding is used

Check the following code:

```php
$this->app->extend(ContainerManager::class, function ($model, $app) {
    if (env('APP_ENV') == 'local')
        return new DockerContainerManager($model);
    else
        return new KubernetesManager($model);
});
```

Imagine that **ContainerManager** will be used by developers to instantiate new **Docker Containers** as the environment is **local** but when the software is running on production then **KubernetesManager** will be used. 

### How to resolve

There are different two different ways to resolve instances:

#### Using make method

That's is only available when **app** variable is on context.

```php
$containerManager = $app->make('ContainerManager');
```

It supports variables using **With** method extension.

```php
$containerManager = $app->makeWith('ContainerManager', ['region' => 'CL1']);
```

#### Using resolve method

This method is very simple because it uses **resolve** helper.

```php
$containerManager = resolve('ContainerManager', ['region' => 'CL1']);
```

#### Using tagged method

This way is perfect to use when you have multiple binds and you wanna aggregate their usages:

```php
$this->app->bind('ServerManager', function () { ... });

$this->app->bind('ClusterManager', function () { ... });

$this->app->bind('DatabaseManager', function () { ... });

$this->app->tag(['ServerManager', 'ClusterManager', 'DatabaseManager'], 'infrastructure');

$this->app->bind('InfrastructureManager', function ($app) {
    return new InfrastructureManager($app->tagged('infrastructure'));
});
```

That's nice.

#### Using PSR-11 way

PSR-11 is a PHP-FIG standard. That's very simple to use. Just automatically inject the **ContainerInterface** and use **get** method.

```php
use Psr\Container\ContainerInterface;

Route::get('/', function (ContainerInterface $container) {
    $containerManager = $container->get('ContainerManager');
});
```


#### Optional: Tracking events

Events are a nice part of Laravel framework. To catch events you should use the following code:

```php
$this->app->resolving(ContainerManager::class, function ($manager, $app) {
    $manager->driver = new KubernetesDriver();
    Logger::info('New container manager instance was requested and is ready to use!');
});
```
