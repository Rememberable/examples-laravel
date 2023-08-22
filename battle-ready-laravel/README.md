For more info, go to DIVEDEEP.md documentation

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

[Documentation](https://phpinsights.com/)

```shell
composer require nunomaduro/phpinsights --dev
```

```shell
php artisan insights
```

```shell
php artisan insights --fix
```

### Enlightn

[Documentation](https://www.laravel-enlightn.com/)

```shell
composer require enlightn/enlightn
```

```shell
php artisan vendor:publish --tag=enlightn
```

```shell
php artisan enlightn
```

### Larastan

```shell
composer require nunomaduro/larastan --dev
```

```shell
touch phpstan.neon
```

```shell
./vendor/bin/phpstan analyse
```

### Style CI

[Documentation](https://styleci.io/)

### Code Coverage

```shell
php artisan test --coverage
```

## Manual Auditing

Pinning the blame on someone won't benefit anyone, so remember to keep an open mind and be open to conversation.

### Investigating "raw" Database Queries

### Finding Incorrect Authorisation

#### Missing Server-Side Authorisation

#### Not Checking the Tenant/Team/Owner

#### Not Scoping the Relationships

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

### Finding "Fake Facades"

### Finding Business Logic in Helpers

### Finding N+1 Queries

#### What Are N+1 Queries?

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

```php
Model::preventLazyLoading(! app()->isProduction());
```

### Finding Controllers That Use Other Controllers

### Finding Hard-Coded Credentials

#### Hard-Coded Emails

#### Hard-Coded API Keys

### Check Open Package Routes

```shell
php artisan route:list --only-vendor
```

### Reviewing Project Documentation

#### Local development instructions

#### Information about the production environment

#### Code style

#### Code standards

#### Frequently asked questions


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

### Choosing What To Test

### Test Structure

"Arrange-Act-Assert" pattern

### Data Providers

## Writing the Tests

### Prioritising Mission-Critical Tests First

### Writing the Rest of the Tests

### Benefits of Writing the Easy Tests First

### Preventing Test Fatigue

## Testing Your UI with Laravel Dusk

## Creating a CI Workflow Using GitHub Actions

### Using an .env.ci File

### Running the Test Suite

### Larastan

# Custom

## Sail

[Documentation](https://laravel.com/docs/sail)

```shell
composer require laravel/sail --dev
```

```shell
php artisan sail:install
```

```shell
./vendor/bin/sail up
```

## Pint

[Documentation](https://laravel.com/docs/pint)

```shell
composer require laravel/pint --dev
```

```shell
./vendor/bin/pint
```

## Debugbar for Laravel

[Documentation](http://phpdebugbar.com/)

```shell
composer require barryvdh/laravel-debugbar --dev
```

## Livewire

[Documentation](https://laravel-livewire.com/)

```shell
composer require livewire/livewire
```
