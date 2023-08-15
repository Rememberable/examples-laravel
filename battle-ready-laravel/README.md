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