# Techniques

### Install laravel debugger for dev
```shell
composer require barryvdh/laravel-debugbar --dev
```

## Measuring Your Database Performance
* Without with() eloquent will do several query db. with() - good optimization!

<table><tr><td> Bad </td> <td> Good </td></tr><tr><td>

```php
<?php
// Bad
$users = User::query()
    ->orderBy('name')
    ->simplePaginate();
```
</td><td>

```php
<?php
// Good
$users = User::query()
    ->with('company')
    ->orderBy('name')
    ->simplePaginate();
```
</td></tr></table>

* You can add index to increase your speed from `20 ms` to `2 ms` in migration `2020_05_30_154756_add_name_index_to_users.php`
```php
<?php
public function up()
{
    Schema::table('users', function (Blueprint $table) {
        $table->index('name');
    });
}
```

## Minimize Memory Usage
<table><tr><td> Bad </td> <td> Good </td></tr><tr><td>

```php
<?php
// Bad
$years = Post::query()
    ->with('author')
    ->latest('published_at')
    ->get()
    ->groupBy(fn ($post) => $post->published_at->year);
```
</td><td>

```php
<?php
// Good
$years = Post::query()
    ->select('id', 'title', 'slug', 'published_at', 'author_id')
    ->with('author:id,name')
    ->latest('published_at')
    ->get()
    ->groupBy(fn ($post) => $post->published_at->year);
```
</td></tr></table>

## Getting One Record From a Has-Many Relationship
* Don't use logic and query in blade, make it as sub select in controller
<table><tr><td> Bad </td> <td> Good </td></tr><tr><td>

```php
<?php
// Bad
{{ $user->logins()->latest()->first()->created_at->diffForHumans() }}
$users = User::query()
    ->with('logins')
    ->orderBy('name')
    ->paginate();
```
</td><td>

```php
<?php
// Good
$users = User::query()
    ->withLastLoginAt()
    ->orderBy('name')
    ->paginate();

public function scopeWithLastLoginAt($query)
{
    $query->addSelect(['last_login_at' => Login::select('created_at')
            ->whereColumn('user_id', 'users.id')
            ->latest()
            ->take(1)
        ])
        ->withCasts(['last_login_at' => 'datetime']);
}
```
</td></tr></table>

## Dynamic Relationships Using Sub-queries
<table><tr><td> Not correct </td> <td> Good </td></tr><tr><td>

```php
<?php
// Not correct
$users = User::query()
    ->withLastLogin()
    ->withLastLogiIpAddressn()
    ->orderBy('name')
    ->paginate();
    
public function scopeWithLastLoginAt($query)
{
    $query->addSelect(['last_login_at' => Login::select('created_at')
            ->whereColumn('user_id', 'users.id')
            ->latest()
            ->take(1)
        ])
        ->withCasts(['last_login_at' => 'datetime']);
}
public function scopeWithLastLoginIpAddress($query)
{
    $query->addSelect(['last_login_ip_address' => Login::select('created_ip_address')
            ->whereColumn('user_id', 'users.id')
            ->latest()
            ->take(1)
        ])
}
```
</td><td>

```php
<?php
// Good
$users = User::query()
    ->withLastLogin()
    ->orderBy('name')
    ->paginate();

public function lastLogin()
{
    return $this->belongsTo(Login::class);
}

public function scopeWithLastLogin($query)
{
    $query->addSelect(['last_login_id' => Login::select('id')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->take(1),
    ])->with('lastLogin');
}
```
</td></tr></table>

## Calculate Totals Using Conditional Aggregates
<table><tr><td> Bad </td> <td> Good </td></tr><tr><td>

```php
<?php
// Bad
$statuses = (object) [];
$statuses->requested = Feature::where('status', 'Requested')->count();
$statuses->planned = Feature::where('status', 'Planned')->count();
$statuses->completed = Feature::where('status', 'Completed')->count();
```
</td><td>

```php
<?php
// Good
if (config('database.default') == 'mysql' || config('database.default') == 'sqlite') {
    $statuses = Feature::toBase()
        ->selectRaw("count(case when status = 'Requested' then 1 end) as requested")
        ->selectRaw("count(case when status = 'Planned' then 1 end) as planned")
        ->selectRaw("count(case when status = 'Completed' then 1 end) as completed")
        ->first();
}

if (config('database.default') == 'pgsql') {
    $statuses = Feature::toBase()
        ->selectRaw("count(*) filter (where status = 'Requested') as requested")
        ->selectRaw("count(*) filter (where status = 'Planned') as planned")
        ->selectRaw("count(*) filter (where status = 'Completed') as completed")
        ->first();
}
```
</td></tr></table>

## Calculate Totals Using Conditional Aggregates
```php
<?php
// In model Comment we have method
public function isAuthor()
{
    return $this->feature->comments->first()->user_id == $this->user_id;
}
```
<table><tr><td> Bad </td> <td> Good </td></tr><tr><td>

```php
<?php
// Bad
$feature->load('comments.user');
```
</td><td>

```php
<?php
// Good
$feature->load('comments.user');
$feature->comments->each->setRelation('feature', $feature);
```
</td></tr></table>
