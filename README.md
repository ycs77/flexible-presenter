# Flexible Presenter

![Latest Version on Packagist](https://img.shields.io/packagist/v/AdditionApps/flexible-presenter)
![GitHub Workflow Status](https://img.shields.io/github/workflow/status/AdditionApps/flexible-presenter/Run%20tests?label=Tests)
[![StyleCI](https://github.styleci.io/repos/243743103/shield?branch=master)](https://github.styleci.io/repos/243743103)

**Easily define just the right data for your Inertia views (or anywhere else you want to, uh, flexibly present).**

This package allows you to define presenter classes that take logic involved in getting data ready for your view layer out of your controller.  It also provides an expressive, fluent API that allows you to modify and reuse your presenters on the fly so that you're only ever providing relevant data.

This package was built specifically for use with [Inertia](https://inertiajs.com) (:heart_eyes:) because we didn't like the fact we were sending more data than we needed to our views.  That said, you're free to use it however you please - it's not dependant on Inertia in any way.

## Installation

You can install the package via composer:

```bash
composer require additionapps/flexible-presenter
```

## Usage

The package includes an artisan command to create a new presenter:

```bash
php artisan make:presenter PostsPresenter
```

This presenter will have the `App\Presenters` namespace and will be saved in `app/Presenters`.

You can also specify a custom namespace, say, `App\Blog`

```bash
php artisan make:presenter "Blog/Presenters/PostsPresenter"
```
This presenter will have the `App\Blog\Presenters` namespace and will be saved in `app/Blog/Presenters`.

### Defining values

A presenter is a class in which you can define all the possible fields that you might want to expose to your Inertia views.  When you call the class from a controller method you can use methods such as `only` or `except` to define which of these fields you want to expose in that given context. More on the API in a bit.

The only required method in a presenter class is `values()` which should return an array with **all** the possible fields you might want to display in a view.  These could simply be values directly on your model.  Note that you can access model properties directly from the `$this` variable, just as you can when using [Laravel API Resources](https://laravel.com/docs/eloquent-resources).

For now, here's a simple presenter class:

```php
class PostPresenter extends FlexiblePresenter
{
    public function values()
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'body' => $this->body,
        ];
    }
}
```

Once a presenter is defined, you are free to return it as part of an Inertia response or in any other context that you need the presented data:

```php
class PostController extends Controller
{
    public function show(Post $post)
    {
        return Inertia::render('Posts/Show', [
            'post' => PostPresenter::make($post)->get(),
        ]);
    }
}
```

### Lazy evaluation

You can also define fields that are computed in more complex ways - for example you may want to add a presenter value that is derived from some other data like a relationship.  In these situations it can be handy to wrap your value definition in a closure.  Doing so will ensure that the value is only computed if it's actually asked for:

```php
public function values()
{
    return [
        'id' => $this->id,
        'comment_count' => function () {
            return $this->comments->count();
        },
    ];
}
```

If you are using PHP >= 7.4 you can make things a little more readable by using the new short closure syntax:

```php
[
    'comment_count' => fn() => $this->comments->count(),
];
```

Now if we call `PostPresenter::make($post)->only(['id'])` the `comment_count` value will not be evaluated meaning we do not need to worry about ensuring that relationship is loaded on the model.

### Nested presenters

If necessary you can define a value that returns another presenter.  This is handy if you want to only return certain values for a relation of the model you're working with:

```php
public function values()
{
    return [
        'id' => $this->id,
        'comments' => function () {
            return CommentPresenter::collection($this->comments)
                ->except('body', 'updated_at');
        },
    ];
}
```

Note that in the example above we're using lazy evaluation so that we don't have to worry about the `comments` relationship being loaded on the `Post` model.  If you know this relationship will always be loaded you could dispense with the closure.

### Instantiating presenters

Once you have defined your presenter class you can use it in your controller (or elsewhere) in your application.  There are three methods for creating a new presenter instance:

#### `PostPresenter::make($post)`

The `make` method accepts a single resource as a parameter.  In the majority of cases this will be an Eloquent model but there is no requirement that you pass a model specifically.  For instance you could pass a some other object or even an associative array and then within the `values()` method in your presenter use it like so:

```php
public function values()
{
    return [
        'id' => $this->resource['id'],
        'title' => $this->resource['title'],
    ];
}
```

#### `PostPresenter::collection($posts)`

The `collection` method accepts an Eloquent collection (or a plain array) of resources as a parameter.  Each member of the collection will be transformed by the presenter as specified.  Again the members of that collection can be Eloquent models, other objects or arrays:

```php
$posts = PostPresenter::collection(Post::all())
    ->only('id', 'title')
    ->get();
```

**Using Pagination**

As well as passing an Eloquent collection or array, you can also pass a Laravel paginator instance.  You are free to pass an instance of either `Illuminate\Pagination\Paginator` or `Illuminate\Pagination\LengthAwarePaginator`. You can also pass a custom paginator as long as it extends the `Illuminate\Pagination\AbstractPaginator` class and implements the `Illuminate\Contracts\Support\Arrayable` interface.

Here's an example of a presenter used with an eloquent collection using simple pagination:

```php
$posts = PostPresenter::collection(Post::simplePaginate())
    ->only('id', 'title')
    ->get();
```

Which will output:

```php
[
    'current_page' => 1,
    'data' => [
        [
            'id' => 1,
            'title' => 'foo',
        ],
        [
            'id' => 2,
            'title' => 'bar',
        ],
    ],
    'first_page_url' => 'http://example.com/list?page=1',
    'from' => 1,
    'next_page_url' => null,
    'path' => 'http://example.com/list',
    'per_page' => 2,
    'prev_page_url' => null,
    'to' => 2,
];
```

#### `PostPresenter::new()`

The `new` method accepts no parameters.  This method is useful for when you have no resource or collection to pass into your presenter (perhaps because the presenter itself is responsible for gathering the resources it needs).

If one of your presenters has a key should return a presented relation as a value, you can use the convenience method `whenLoaded` to conditionally include the relation:

```php
// In PostPresenter
return [
    'comments' => CommentPresenter::collection($this->whenLoaded('comments')),
];
```

In the above example the `comments` key will be a collection of presented comments if the relation is loaded and `null` if they are not.

### Configuring presenters

With a new presenter instance you are now free to configure it in whatever way makes sense for your the current context.  All the following methods can be chained onto `make()`, `collection()` or `new()`.

#### `only()`

The `only` method accepts keys that you want to return from your presenter instance.  You can pass these keys as an array (`->only(['id', 'title'])`) or as individual string arguments (`->only('id', 'title')`).

#### `except()`

The `except` method accepts keys that you want to exclude from your presenter instance.  You can pass these keys as an array (`->except(['id', 'title'])`) or as individual string arguments (`->except('id', 'title')`).

#### `with()`

The `with` method accepts a closure that takes as it's only argument the resource that is being transformed.  You can use the `with` method to overwrite the values of existing keys or to add entirely new keys to your presenter on-the-fly.  It should return an array with the keys you want to overwrite/add along with their values.

```php
PostPresenter::make($post)->with(function($post){
    return [
        'title' => strtoupper($post->title),
        'new_key' => 'Some value',
    ];
});
```

#### `preset()`

You may find that there are combinations of presenter values that you use repeatedly throughout your application.  If so, rather than explicitly asking for those particular fields each time you can add a 'preset' to your presenter class and ask for that preset instead:

```php
PostPresenter::make($post)->preset('summary');
```

Within your presenter class you should create a method with the name of your preset, prefixed with 'preset'.  Given the example above we would create a method like this:

```php
public function presetSummary()
{
    return $this->only('title', 'published_at');
}
```

Your preset method should use the same presenter API methods above to build up a set of values.  Note that all of these API methods can be chained, so, for example, you can modify a preset on the fly if you wish:

```php
PostPresenter::make($post)->preset('summary')->with(function () {
    return [
        'comment_count' => $post->comments->count(),
    ];
});
```

**Caveat: repeated method calls**

Just bear in mind that if you use an API method in your preset method (for example `only`) and then chain another `only` method onto it when using your presenter, that the last call to `only` will be the one to take effect:

```php
// In PostPresenter...
public function presetSummary()
{
    return $this->only('title', 'body');
}

// In Controller...
PostPresenter::make($post)->preset('summary');
// Will return ['title' => 'foo', 'body' => 'bar']

PostPresenter::make($post)->preset('summary')->only('id');
// Will return ['id' => 1]
```

#### `appends()`

If you are presenting a paginated collection of resources you may want to add some additional key/value pairs to the outer array that wraps your data.  To do this, you can use the `appends()` method to specify the keys and values you wish to set.  Let's say you're presenting a custom paginator instance that produces this output:

```php
[
    'current_page' => 1,
    'data' => [
        // your presented resources
    ],
    'first_page_url' => 'http://example.com/list?page=1',
    'from' => 1,
    'next_page_url' => null,
    'path' => 'http://example.com/list',
    'per_page' => 2,
    'prev_page_url' => null,
    'to' => 2,
    'links' => [
        'create' => 'some-url',
    ],
];
```

Using appends you are free to add (or overwrite) the keys in this array:

```php
$posts = PostPresenter::collection($paginatedCollection)
    ->only('id')
    ->appends([
        'foo' => 'bar',
        'links' => ['store' => 'some-other-url'],
    ])
    ->get();
```

This will now output:

```php
[
    'current_page' => 1,
    'data' => [
        // your presented resources
    ],
    'first_page_url' => 'http://example.com/list?page=1',
    'from' => 1,
    'next_page_url' => null,
    'path' => 'http://example.com/list',
    'per_page' => 2,
    'prev_page_url' => null,
    'to' => 2,
    'foo' => 'bar',
    'links' => [
        'create' => 'some-url',
        'store' => 'some-other-url',
    ],
];
```

Note that appended values are merged recursively (as in the `links` example above).

### Returning values

Once you've configured an instantiated presenter the way you want, you can get the values as an array by chaining the `get` method:

```php
PostPresenter::make($post)->only('id', 'title')->get();
```

If you wish to return all the defined values in your presenter (without any configuration) then you can use the `all` method:

```php
PostPresenter::make($post)->all();
```

This is equivalent to the following:

```php
PostPresenter::make($post)->get();
```

Finally, all presenters implement the `Arrayable` interface, so you are passing your presenter to a context that looks for this contract, your presenter values will be automatically converted to an array without you having to use `get()` (or `all()`).  

Here's an example using an Inertia response:

```php
return Inertia::render('Posts/Show', [
    'post' => PostPresenter::make($post)->only('title'),
]);
```
In the example above we're defining our Inertia props manually in an array.  You can also pass a presenter instance directly - it will be converted into an array of props automatically:

```php
return Inertia::render('Posts/Show', PostPresenter::make($post)->only('title'));
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) for details.

## Security

If you discover any security related issues, please email john@addition.com.au instead of using the issue tracker.

## Treeware

You're free to use this package, but if it makes it to your production environment we would highly appreciate you buying or planting the world a tree.

It’s now common knowledge that one of the best tools to tackle the climate crisis and keep our temperatures from rising above 1.5C is to plant trees. If you contribute to my forest you’ll be creating employment for local families and restoring wildlife habitats.

You can buy trees at [Offset Earth](https://offset.earth/treeware)

Read more about Treeware at [Treeware](https://treeware.earth)

## Credits

John Wyles

All Contributors

## License

The MIT License (MIT). Please see [License File](LICENSE.md) File for more information.
