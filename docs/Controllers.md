# Controllers

## Defining Controllers

To define your controllers you should use the next syntax:

```php
class WelcomeController extends \App\Http\Controllers\Controller {
    public function index() { return view('welcome'); }
}
```

Of course, you should register them into the **web.php** or **api.php** as the next way:

```php
Route::get('/', 'WelcomeController@index');
```

You can see that code and the usage of the **class name** and **method name** of your controller.

## Controller Namespacing

Namespacing as all classes on PHP, are the virtual logic folder of your code. 

When you are registering controllers you normally use the next syntax:

```php
Route::get('/', 'Path\WelcomeController@index');
```

Then the **WelcomeController** is located on **App\Http\Controllers\Path**. Usually defined on the head of file:

```php
<?php
namespace App\Http\Controllers\Path;
use App\Http\Controllers\Controller;
class WelcomeController extends Controller {}
```

Well done, you have learned how to use **controller namespacing**.
