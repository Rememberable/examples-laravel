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


### Checking Validation

#### Applying the Basic Rules

- Is the field required?
- What data type is the field?
- Is there a minimum value (or length) this field can be?
- Is there a maximum field (or length) this field can be?

So you can think of your validation rules as being written using the following format:
```text
REQUIRED|DATATYPE|MIN|MAX|OTHERS
```

#### Checking for Empty Validation

This means that the twitter_handle field can now be passed in the request and stored. This can be a dangerous move 
because it circumvents the purpose of validating the data before it is stored.

```php
$validated = $request->validate([
    'name' => 'required|string|max:255',
    'email' => 'required|email',
    'twitter_handle' => '',
]);
```

### Finding "Fake Facades"

```php
class PricingService
{
    private static bool $withTax = false;
 
    public static function calculatePrice(Product $product)
    {
        return static::$withTax
            ? $product->price * 1.2
            : $product->price;
    }
 
    public static function withTax(bool $withTax = true): void
    {
        static::$withTax = $withTax;
    }
}
```

```php
PricingService::withTax();
$priceWithTax = PricingService::calculatePrice($productOne);

$priceWithoutTax = PricingService::calculatePrice($productTwo);
```

Although it appears that the priceWithoutTax variable should have had the price calculated without the tax, it will 
have been included because we called the withTax method beforehand.

### Finding Business Logic in Helpers

```php
use App\DataTransferObjects\UserData;
use App\Jobs\AlertNewUser;
use App\Models\Role;
use App\Models\User;
use Illuminate\Support\Facades\DB;

if (! function_exists('store_user')) {
    function store_user(UserData $userData): User
    {
        return DB::transaction(function () use ($userData) {
            $user = User::create([
                'email' => $userData->email,
            ]);
                
            $user->roles()->attach(
                Role::where('name', 'general')->first()
            );
            
            AlertNewUser::dispatch($user);
            
            return $user;
        });
    }
}
```

One of the easiest ways of solving this issue is to move the functions into classes. This would provide us with the 
ability to resolve the class from the service container so that we can mock it during our tests or replace it with a 
test double.

### Finding N+1 Queries

#### What Are N+1 Queries?

```php
$comments = Comment::all();

foreach ($comments as $comment) {
    print_r($comment->author->name);
}
```

Fix N+1 quires:
```php
$comments = Comment::with('author')->get();

foreach ($comments as $comment) {
    print_r($comment->author->name);
}
```

#### How to Disable N+1 Queries

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        Model::preventLazyLoading();
    }
}
```

Do not crush production:
```php
Model::preventLazyLoading(! app()->isProduction());
```

### Finding Controllers That Use Other Controllers

```php
class UserController extends Controller
{
    public function index()
    {
        return User::where('is_admin', false)
            ->get()
            ->toArray();
    }
}
```
```php
class AdminUserController extends Controller
{
    public function index()
    {
        return User::where('is_admin', true)
            ->get()
            ->toArray();
    }
}
```

```php
class AllUserController extends Controller
{
    public function index()
    {
        $nonAdminUsers = (new UserController())->index();
        $adminUsers = (new AdminUserController())->index();
 
        return [
            'users' => $nonAdminUsers,
            'admin_users' => $adminUsers,
        ];
    }
}
```

### Finding Logic and Queries in Blade Views

```php
@php
    $users = User::where('is_admin', false)
    ->select()
    ->withCount(['posts' => function ($query) {
        $query->where('published_at', '<=', now());
    }])
    ->get();
@endphp

<table>
    <tr>
        <th>Name</th>
        <th>Posts</th>
    </tr>
    @foreach($users as $user)
        <tr>
            <td>$user->name</td>
            <td>$user->posts_count</td>
        </tr>
    @endforeach
</table>
```

### Finding Hard-Coded Credentials

#### Hard-Coded Emails

```php
protected function gate()
{
    Gate::define('viewTelescope', function ($user) {
        return in_array($user->email, [
            'mail@example.com',
            'joe.bloggs@example.com',
        ]);
    });
}
```

Fix:
```dotenv
TELESCOPE_ALLOWED_EMAILS="mail@example.com,joe.bloggs@example.com"
```
```php
// config/telescope.php file
return [
    // …
    'allowed_emails' => env('TELESCOPE_ALLOWED_EMAILS', ''),
    // …
];
```
```php
protected function gate()
{
    Gate::define('viewTelescope', function ($user) {
        return in_array(
            $user->email,
            explode(',', config('telescope.allowed_emails')),
        );
    });
}
```

#### Hard-Coded API Keys

On some projects, this key was hard-coded as a const or property at the top of the code using it.

### Check Open Package Routes

This will result in only the routes being displayed that have been registered by packages included in your project.

```shell
php artisan route:list --only-vendor
```

### Reviewing Project Documentation

Up-to-date documentation can make it much easier for new developers to be onboarded onto a project.

#### Local development instructions

What environment are they expected to use? Docker? Valet? Homestead?

These instructions are also useful for telling them about any API keys that they might need to configure, or any 
commands that they need to run

#### Information about the production environment

What sort of infrastructure is being used to host the project? Is it running on an AWS EC2? Is it a serverless-based 
application that's using Laravel Vapor? Is Laravel Octane being used? What type of database are you using? In some 
cases the specifics of this don't matter much(and could compromise security if seen by the wrong people)

#### Code style

What code style should the developers use? You might be using a common standard, such as PSR-12, or you might have your 
own company style guide

#### Code standards

You may also want to specify any standards that you expect the developers to follow.

#### Frequently asked questions

As your project and team grow, you might find questions commonly asked about the project by new developers.


# Testing

## Planning Your Testing Strategy

This will make it easier to build your new test suite, and will also make it easier to cooperate with your peers if
working as part of a team.

## The Benefits of Writing Tests

Writing tests before the code is released into production is almost always the best option!

### Spotting Bugs Early

How many times have you written code, ran it once or twice, and then committed it?

### Making Future Work and Refactoring Easier

Without tests, how will you know that changing or adding code isn't going to break the existing functionality?

### Changing the Way You Approach Writing Code

To write code in a way that can be tested, you have to look at the structure of your classes and methods from a 
slightly different angle than before.

### Tests-As-Documentation

Your tests can act as a form of documentation and give hints that can help a developer understand what a method does.

### Prove That Bugs Exist

In the event that a bug is reported by a user, you can write tests to mimic the actions of the user.

## Structuring Your Tests

When writing your tests, it's important that you define a structure before starting.

### Directory Structure

There are different directory structures you can use for writing tests, and every project that I've worked on has 
always been different

`app/Http/Controllers/UserController@index` -> `tests/Feature/Http/Controllers/UserController/IndexTest.php`

`app/Http/Controllers/UserController@store` -> `tests/Feature/Http/Controllers/UserController/StoreTest.php`

`app/Http/Controllers/UserController@update` -> `tests/Feature/Http/Controllers/UserController/UpdateTest.php`

Of course, you don't need to follow this same structure, and you can choose whatever works best for you and your team. 
But I do think it's important that you and your team decide on a suitable structure that is consistent.

### Choosing What To Test

If you're new to writing tests, you may find that one of the most difficult parts of the process is figuring out what 
to actually test.

When I'm writing a test, I try to break the method's code down and identify specific pieces that I can assert against.
For example, we might want to assert against:

- The returned output of a method based on the input.
- Whether a user was authenticated or logged out.
- Whether any exceptions were thrown.
- Whether any rows in the database changed.
- Whether any jobs were added to the queue.
- Whether any events were dispatched.
- Whether any mail or notifications were sent (or queued).
- Whether any external HTTP calls were made

```php
use App\DataTransferObjects\UserData;
use App\Models\User;
use App\Services\MyThirdPartyService;

public function store(UserData $userData): User
{
    return DB::transaction(function () use ($user, $userData) {
        $user = User::create([
            'email' => $userData->email,
        ]);

        $this->assignRole('admin');
 
        $service = app(MyThirdPartyService::class);
 
        if (! $service->createUser($user)) {
            throw new \Exception('User could not be created');
        }
 
        return $user;
    });
}
```

```php
/** @test */
public function user_can_be_created()
{
    // 1. Assert that the user is created in the database.
    // 2. Assert that the user is assigned the 'admin' role.
    // 3. Assert that the user is created in the third-party
    // system.
    // 4. Assert that the method returns the created model.
}

/** @test */
public function error_is_thrown_if_user_cannot_be_created()
{
    // 1. Assert that an exception is thrown if the user
    // cannot be created in the third-party system.
}
```

### Test Structure

"Arrange-Act-Assert" pattern

**Arrange** - Make all the preparations for your test to start. This might mean using a
model factory to add data to your database. It might include you creating any mocks,
test doubles, or dummy files.

**Act** - Act on the target behaviour. This might mean making an HTTP call to a
controller or calling a method in a class.

**Assert** - Assert against the results of the target and the expected outcomes. For
example, assert that the correct data was returned from the tested method or that a
new row was created or updated in the database.

```php
use App\Models\Post;

class BlogController extends Controller
{
    public function index()
    {
        $posts = Post::query()
            ->where('published_at', '<=', now())
            ->orderByDesc('published_at')
            ->get();
 
        return view('blog.index', $posts);
    }
}
```

To "arrange" the test we'd first need to:
- Create some published blog posts in our database.
- Create some unpublished blog posts in our database.


We'd then want to "act" on the test by:
- Making an HTTP call to the controller method (which we'll assume can be reached using a route with the name blog.index).


We can then "assert" in our test by asserting that:
- The correct view is returned.
- The published blog posts are returned in order of their published_at date.
- The unpublished blog posts are not returned.

As a result, our test may look something like so:

```php
namespace Tests\Feature\Controllers\BlogController;

use App\Post;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Database\Eloquent\Factories\Sequence;
use Illuminate\Foundation\Testing\LazilyRefreshDatabase;
use Tests\TestCase;

class IndexTest extends TestCase
{
    use LazilyRefreshDatabase;
 
    /** @test */
    public function view_is_returned(): void
    {
        // ARRANGE...
        // Create a post that has not been published yet.
        // This should not be included in the response.
        Post::factory()
            ->create(['published_at' => now()->addDay()]);

        // Create 3 posts that have been published.
        // These should be included in the response.
        $posts = Post::factory()
            ->count(3)
            ->state(new Sequence(
                ['published_at' => now()->subYear()],
                ['published_at' => now()->subMinute()],
                ['published_at' => now()->subSecond()],
            ))
            ->create();
 
        // ACT...
        $response = $this->get(route('blog.index'));
 
        // ASSERT...
        $response->assertOk()
            ->assertViewIs('blog.index')
            ->assertViewHas(
                'posts',
                function (Collection $viewPosts) use ($posts) {
                    // Assert that only the published posts were
                    // returned and that they were in the
                    // correct order.
                    $expectedPosts = [
                        $posts[2]->id,
                        $posts[1]->id,
                        $posts[0]->id,
                    ];
 
                    return $viewPosts->pluck('id')->toArray() === $expectedPosts;
                }
            );
    }
}
```

### Data Providers

You may find when writing tests that you want to test the same method but with many different inputs or scenarios. 
This can lead to many tests that are mostly identical; the maintenance of these tests can be cumbersome and tedious.

```php
use Illuminate\Foundation\Testing\WithFaker;
use Tests\TestCase;

class CalculateReadTimeTest extends TestCase
{
    use WithFaker;
 
    /**
    * @test
    * @dataProvider readTimeDataProvider
    */
    public function correct_read_time_is_returned(int $wordCount, int $expectedResult): void
    {
        $words = $this->faker->words($wordCount, true);
 
        self::assertEquals($expectedResult, calculate_read_time($words));
    }
 
    public function readTimeDataProvider(): array
    {
        return [
            [3, 1],
            [265, 60],
            [530, 120],
            [500, 114],
            [2000, 453],
        ];
    }
}
```

```php
use Illuminate\Foundation\Testing\WithFaker;
use Tests\TestCase;

class CalculateReadTimeTest extends TestCase
{
    use WithFaker;
 
    /**
    * @test
    * @testWith [3, 1]
    *           [265, 60]
    *           [530, 120]
    *           [500, 114]
    *           [2000, 453]
    */
    public function correct_read_time_is_returned(int $wordCount, int $expectedResult): void
    {
        $words = $this->faker->words($wordCount, true);

        self::assertEquals($expectedResult ,calculate_read_time($words));
    }
}
```

## Writing the Tests

### Prioritising Mission-Critical Tests First

To start testing your project, you'll first want to identify the mission-critical parts of your code. For example, on 
an e-commerce site, you might identify the "basket" and "checkout" features as mission-critical parts of your code.

### Writing the Rest of the Tests

After you've started building up a test suite that covers the mission-critical parts of your project, you can write 
tests for other parts of the system.

### Benefits of Writing the Easy Tests First

A key benefit of writing the easiest tests first is that you can write them faster than tests for more difficult 
methods. As a result, you can write more tests in a shorter space of time and grow your test suite faster.

### Preventing Test Fatigue

As you write more tests, you'll feel more comfortable writing them and find you can write them more quickly.

## Testing Your UI with Laravel Dusk

UI tests can simulate user actions on your site or application.

### Installation

```shell
composer require --dev laravel/dusk
```

```shell
php artisan dusk:install
```

### Running Failed Tests and Groups

```shell
php artisan dusk:fails
```

```php
/**
* @group authentication
* @test
*/
public function user_can_login_with_2fa()
{
    // ...
}
```

```shell
php artisan dusk --group=authentication
```

## Creating a CI Workflow Using GitHub Actions

Once you have a test suite set up and working, you'll want to set up a CI (continuous integration) workflow that can be 
used to run checks against all our future code.

### Using an .env.ci File

You can place all of your environment variables here, and when we start our GitHub Action, we'll copy our `.env.ci` file 
so it becomes our `.env` file while the tests are running.

### Running the Test Suite

In `.github/workflows/test.yml` file

For Mysql:

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: testing
    ports:
      - 33306:3306
    options: --health-cmd="mysqladmin ping" --healthinterval=10s --health-timeout=5s --health-retries=3
```

For PostgreSQL:

```yaml
services:
  postgres:
    image: postgres:14.5
    env:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
      POSTGRES_DB: testing
    ports:
      - 33306:3306
  options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
```

### Larastan

In `.github/workflows/larastan.yml` file


# Custom

## Sail

Laravel Sail is a light-weight command-line interface for interacting with Laravel's default Docker development environment.

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

## Debugbar for Laravel

Package for Laravel Debugging

[Documentation](http://phpdebugbar.com/)

Installation
```shell
composer require barryvdh/laravel-debugbar --dev
```
