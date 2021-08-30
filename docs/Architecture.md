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

## Service Providers

Service providers are a simple way to add features to your application not inside of **Controllers**, **Commands** or **Repositories**.

Every provider should be extend the ServiceProvider class of Illuminate\Support namespace and must be have **register** and **boot** methods.

The provider can be created using **make:provider** command of **artisan** CLI. 

The **register** method should be used only to bind things to app container.

The **boot** method will be executed after the registration of the provider.

```php
class MyProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton(ServerManager::class, function ($app) {
            return new ServerManager(config('region'), config('server_provider_token'));
        });
    }

    public function boot()
    {
        $manager = $this->app->resolve(ServerManager::class);
        $manager->initialize();
    }   
}
```

Also you can deffer the binding implementing **DeferrableProvider** and that's very useful do not bind objects or classes when that is not needed. To reach this goal you should implement your provider as the following way:

```php
use Illuminate\Contracts\Support\DeferrableProvider;
use Illuminate\Support\ServiceProvider;

class MyProvider extends ServiceProvider implements DeferrableProvider {
    public function register()
    {
        $this->app->singleton(ServerManager::class, function ($app) {
            return new ServerManager(config('region'), config('server_provider_token'));
        });
    }
    public function provides()
    {
        return [ServerManager::class];
    }
}
```

Your binding will be done only when is needed by implementing **DeferrableProvider** and **provides** method. 

As is commonly, the provider should be registered on **providers** array of **config/app.php** config.

ServiceProviders also support **Automatic Injection**. 

The **AppServiceProviders** has two arrays named **bindings** and **singletons** to provide a simple way to bind implementations to classes.

## Facades

Facades are simple ways to add features to your application using service container.

Let's make a example:


```php
class Logger extends Facade {
    protected static function getFacadeAccessor() { return 'logger'; }
}
```

So if the service container via service provider has bind a class or object in **logger** space then you will retrieve that instance using the facade.

```php
Logger::infoToSlack("Hello from the App team!");
```

Remember that should be bind into a Service Provider and registered in **app.php** config.

Facades support testing out-of-box. So, imagine that **infoToSlack** return a boolean.

Think about a command with the past code then in your tests you can write the next code:

```php
Logger::shouldReceive('infoToLog')
    ->with('message')
    ->andReturn(TRUE);

$this->artisan("run:logger:infoToLog Hello");
```

The command will be send hello to Slack and obviusly the test will be pass.

## HTTP Verbs

As we expect, this topic is really simple:

We have different verbs of HTTP, like GET, HEAD, POST, PUT, PATCH, DELETE, CONNECT, TRACE and OPTIONS. 

In Laravel the supported methods are:

#### GET

Used on **index** or **views** operations.

#### POST

Used on **store** or **insert** operations.

#### PUT or PATCH

Used on **update** operations.

#### DELETE

Used on **destroy** operations.

#### OPTIONS

Used to list **operations**. 
