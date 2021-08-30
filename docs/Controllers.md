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
