# Auditing

## Planning Your Auditing Strategy

### Getting Into the Right Mindset

Just because you would have written a particular feature differently, doesn't necessarily mean that it's wrong.

When we're looking through the code, we're looking for code that:

- Has bugs
- Is vulnerable to security exploits
- Makes testing difficult
- Makes it difficult to extend in the future.

### Recording Your Findings

As you're working through your audit, you'll need to find a place to record your findings so that you can tackle the 
issues later on. 

When I've audited projects that I've been working on by myself, I've used TODOs directly in the code

## Automated Tooling

### PHP Insights

PHP Insights is a tool that has built-in checks that you can use to help make your code reliable, loosely coupled, 
simple, and clean.

[Documentation](https://phpinsights.com/)

Installation
```shell
composer require nunomaduro/phpinsights --dev
```
```shell
php artisan vendor:publish --provider="NunoMaduro\PhpInsights\Application\Adapters\Laravel\InsightsServiceProvider"
```

Analysing Your Code with PHP Insights
```shell
php artisan insights
```

```shell
php artisan insights app/Http/Controllers
```

```shell
php artisan insights app/Http/Controllers/Controller.php
```

Automatically Fixing Issues
```shell
php artisan insights --fix
```

### Enlightn

[Documentation](https://www.laravel-enlightn.com/)

Installation
```shell
composer require enlightn/enlightn
```

Publish the package's config file
```shell
php artisan vendor:publish --tag=enlightn
```

Run Enlightn
```shell
php artisan enlightn
```

### Larastan

Larastan is a package built on top of PHPStan (a static analysis tool) that can be used to perform code analysis.

Installation
```shell
composer require nunomaduro/larastan --dev
```

Create a `phpstan.neon` file in your Laravel project's root.
```shell
touch phpstan.neon
```

Run Larastan
```shell
./vendor/bin/phpstan analyse
```

False Positives. You could either add `// @phpstan-ignore-next-line` above the method call like so:
```php
// @phpstan-ignore-next-line
app(NewsletterService::class)->generateStatsForAdmin(auth('admin')
->user());
```

Larastan provides the ability to set a "baseline", which effectively ignores the existing code and only detects issues 
in new or changed code.
```shell
./vendor/bin/phpstan analyse --generate-baseline
```

### Style CI

[Documentation](https://styleci.io/)

### Code Coverage

Running the tests and seeing the code coverage can also be useful for when you come to deciding on your testing 
strategy later. For example, it can help you determine which parts of the project should be prioritised for writing 
tests.

Generating the Code Coverage Report
```shell
php artisan test --coverage
```

if we want to specify that the application must have at least 70% code coverage, you could run the following command
```shell
php artisan test --coverage --min=70
```

## Manual Auditing

Pinning the blame on someone won't benefit anyone, so remember to keep an open mind and be open to conversation.

### Investigating "raw" Database Queries

The project that I was auditing was a multi-tenant application and provided a search functionality for users using code 
similar to this:
```php
User::where('tenant_id', auth()->user()->tenant_id)
    ->whereRaw(
        "CONCAT(first_name, ' ', last_name) LIKE '%"
        .$request->input('search')."%'"
    )
    ->get();
```

```sql
SELECT * FROM `users` WHERE tenant_id = 1 AND CONCAT(first_name, ' ', last_name) LIKE '%Joe B%'
```

```text
/users?search='OR 1=1;-- --
```

```sql
SELECT * FROM `users` WHERE tenant_id = 1 AND CONCAT(first_name, ' ', last_name) LIKE '%' OR 1=1;-- --'%'
```

```php
User::where('tenant_id', auth()->user()->tenant_id)
    ->whereRaw(
        'CONCAT(first_name, " ", last_name) LIKE ?',
        ['%'.$request->input('email').'%']
    )
    ->get();
```

Doing this makes it much easier to track down any usages of raw queries (such as in `whereRaw`, `orWhereRaw`, 
`selectRaw`, etc.)

### Finding Incorrect Authorisation

A potentially harmful security issue that I have found in many projects that I have audited is incomplete 
(or incorrect) authorisation checks.

#### Missing Server-Side Authorisation

```php
@if(auth()->user()->hasPermissionTo('view-users'))
    <a href="{{ route('users.show', $user) }}">
        View {{ $user->name }}
    </a>
@endif
```

```php
public function show(User $user): View
{
    return view('users.show', [
        'user' => $user,
    ]);
}
```

```php
public function show(User $user): View
{
    if (!$user->hasPermissionTo('view-users')) {
        abort(403);
    }

    return view('users.show', [
        'user' => $user,
    ]);
}
```

```php
class UserPolicy
{
    public function view(User $authUser, User $user): bool
    {
        return $authUser->hasPermissionTo('view-users');
    }
}
```
```php
@if(auth()->user()->can('view', $user))
    <a href="{{ route('users.show', $user) }}">
        View {{ $user->name }}
    </a>
@endif
```

```php
public function show(User $user): View
{
    $this->authorize('view', $user);

    return view('users.show', [
        'user' => $user,
    ]);
}
```


#### Not Checking the Tenant/Team/Owner

```php
class PostPolicy
{
    public function publish(User $user, Post $post): bool
    {
        return $user->hasPermissionTo('publish-post');
    }
}
```

```php
class PostPolicy
{
    public function publish(User $user, Post $post): bool
    {
        return $user->hasPermissionTo('publish-post') && $user->team_id === $post->team_id;
    }
}
```

#### Not Scoping the Relationships

```php
Route::get(
    '/projects/{project}/invoices/{invoice}',
    [InvoiceController::class, 'show']
);
```

```php
public function show(Project $project, Invoice $invoice): View
{
    if ($project->user_id !== auth()->user()->id) {
        abort(403);
    }

    return view('invoices.show', [
        'invoice' => $invoice,
    ]);
}
```

```php
public function show(Project $project, Invoice $invoice): View
{
    if ($project->user_id !== auth()->user()->id) {
        abort(403);
    }
    
    if ($project->user_id !== auth()->user()->id) {
        abort(403);
    }
    
    return view('invoices.show', [
        'invoice' => $invoice,
    ]);
}
```

```php
Route::get(
    '/projects/{project}/invoices/{invoice}',
    [InvoiceController::class, 'show']
)->scopeBindings();
// This removes the need for the `$invoice->project_id !== $project->id` check in the controller method.
```




# Custom

## Sail

Laravel Sail is a light-weight command-line interface for interacting with Laravel's default Docker development 
environment.

[Documentation](https://laravel.com/docs/sail)

Installation
```shell
composer require laravel/sail --dev
```

Start Sail
```shell
./vendor/bin/sail up
```

## Pint

Laravel Pint is an opinionated PHP code style fixer for minimalists.

[Documentation](https://laravel.com/docs/pint)

Installation
```shell
composer require laravel/pint --dev
```

Start to fix code style
```shell
./vendor/bin/pint
```

Start to inspect your code for style errors without actually changing the files
```shell
./vendor/bin/pint --test
```

If you would like Pint to only modify the files that have uncommitted changes
```shell
./vendor/bin/pint --dirty
```
