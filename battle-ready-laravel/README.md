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
