# Search Feature

## Multi-Column Searching

```php
<?php
$users = User::query()
    ->with('company')
    ->search(request('search'))
    ->paginate();

public function scopeSearch($query, string $terms = null)
{
    if (config('database.default') == 'mysql' || config('database.default') === 'sqlite') {
        collect(explode(' ', $terms))->filter()->each(function ($term) use ($query) {
            $term = '%'.$term.'%';
            $query->where(function ($query) use ($term) {
                $query->where('first_name', 'like', $term)
                    ->orWhere('last_name', 'like', $term)
                    ->orWhereHas('company', function ($query) use ($term) {
                        $query->where('name', 'like', $term);
                    });
            });
        });
    }
    if (config('database.default') == 'pgsql') {
        collect(explode(' ', $terms))->filter()->each(function ($term) use ($query) {
            $term = '%'.$term.'%';
            $query->where(function ($query) use ($term) {
                $query->where('first_name', 'ilike', $term)
                    ->orWhere('last_name', 'ilike', $term)
                    ->orWhereHas('company', function ($query) use ($term) {
                        $query->where('name', 'ilike', $term);
                    });
            });
        });
    }
}
```
* Add index to tables
```php
<?php
Schema::create('users', function (Blueprint $table) {
    $table->string('first_name')->index();
    $table->string('last_name')->index();
});
Schema::create('companies', function (Blueprint $table) {
    $table->string('name')->index();
});
```

## When it Makes Sense to Run Additional Queries

```php
<?php
if (config('database.default') === 'mysql' || config('database.default') === 'sqlite') {
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($term) use ($query) {
        $term = $term.'%';
        $query->where(function ($query) use ($term) {
            $query->where('first_name', 'like', $term)
                ->orWhere('last_name', 'like', $term)
                ->orWhereIn('company_id', Company::query()
                    ->where('name', 'like', $term)
                    ->pluck('id')
                );
        });
    });
}

if (config('database.default') === 'pgsql') {
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($term) use ($query) {
        $term = $term.'%';
        $query->where(function ($query) use ($term) {
            $query->where('first_name', 'ilike', $term)
                ->orWhere('last_name', 'ilike', $term)
                ->orWhereIn('company_id', Company::query()
                    ->where('name', 'ilike', $term)
                    ->pluck('id')
                );
        });
    });
}
```

## Use Unions to Run Queries Independently

```php
<?php
if (config('database.default') === 'mysql' || config('database.default') === 'sqlite') {
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($term) use ($query) {
        $term = preg_replace('/[^A-Za-z0-9]/', '', $term).'%';
        $query->whereIn('id', function ($query) use ($term) {
            $query->select('id')
                ->from(function ($query) use ($term) {
                    $query->select('users.id')
                        ->from('users')
                        ->where('users.first_name', 'like', $term)
                        ->orWhere('users.last_name', 'like', $term)
                        ->union(
                            $query->newQuery()
                                ->select('users.id')
                                ->from('users')
                                ->join('companies', 'users.company_id', '=', 'companies.id')
                                ->where('companies.name', 'like', $term)
                        );
                }, 'matches');
        });
    });
}

if (config('database.default') === 'pgsql') {
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($term) use ($query) {
        $term = preg_replace('/[^A-Za-z0-9]/', '', $term).'%';
        $query->whereIn('id', function ($query) use ($term) {
            $query->select('id')
                ->from(function ($query) use ($term) {
                    $query->select('users.id')
                        ->from('users')
                        ->where('users.first_name', 'ilike', $term)
                        ->orWhere('users.last_name', 'ilike', $term)
                        ->union(
                            $query->newQuery()
                                ->select('users.id')
                                ->from('users')
                                ->join('companies', 'users.company_id', '=', 'companies.id')
                                ->where('companies.name', 'ilike', $term)
                        );
                }, 'matches');
        });
    });
}
```

## Fuzzier Searching With Regular Expressions

* Add index to fields with regular expressions and remove index from common fields
```php
<?php
if (config('database.default') === 'mysql') {
    Schema::create('users', function (Blueprint $table) {
        $table->id();
        $table->foreignId('company_id')->constrained('companies');
        $table->string('first_name');
        $table->string('first_name_normalized')->virtualAs("regexp_replace(first_name, '[^A-Za-z0-9]', '')")->index();
        $table->string('last_name');
        $table->string('last_name_normalized')->virtualAs("regexp_replace(last_name, '[^A-Za-z0-9]', '')")->index();
        $table->string('email')->unique();
        $table->timestamp('email_verified_at')->nullable();
        $table->string('password');
        $table->rememberToken();
        $table->timestamps();
    });
    Schema::create('companies', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->string('name_normalized')->virtualAs("regexp_replace(name, '[^A-Za-z0-9]', '')")->index();
        $table->timestamps();
    });
}

if (config('database.default') === 'pgsql') {
    Schema::create('users', function (Blueprint $table) {
        $table->id();
        $table->foreignId('company_id')->constrained('companies');
        $table->string('first_name');
        $table->string('last_name');
        $table->string('email')->unique();
        $table->timestamp('email_verified_at')->nullable();
        $table->string('password');
        $table->rememberToken();
        $table->timestamps();
        $table->rawIndex("regexp_replace(first_name, '[^A-Za-z0-9]', '')", 'users_first_name_normalized_index');
        $table->rawIndex("regexp_replace(last_name, '[^A-Za-z0-9]', '')", 'users_last_name_normalized_index');
    });
    Schema::create('companies', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->timestamps();
        $table->rawIndex("regexp_replace(name, '[^A-Za-z0-9]', '')", 'companies_name_normalized_index');
    });
}
```

```php
<?php
if (config('database.default') === 'mysql') {
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($term) use ($query) {
        $term = preg_replace('/[^A-Za-z0-9]/', '', $term).'%';
        $query->whereIn('id', function ($query) use ($term) {
            $query->select('id')
                ->from(function ($query) use ($term) {
                    $query->select('users.id')
                        ->from('users')
                        ->where('users.first_name_normalized', 'like', $term)
                        ->orWhere('users.last_name_normalized', 'like', $term)
                        ->union(
                            $query->newQuery()
                                ->select('users.id')
                                ->from('users')
                                ->join('companies', 'users.company_id', '=', 'companies.id')
                                ->where('companies.name_normalized', 'like', $term)
                        );
                }, 'matches');
        });
    });
}

if (config('database.default') === 'sqlite') {
    throw new \Exception('This lesson does not support SQLite.');
}

if (config('database.default') === 'pgsql') {
    collect(str_getcsv($terms, ' ', '"'))->filter()->each(function ($term) use ($query) {
        $term = preg_replace('/[^A-Za-z0-9]/', '', $term).'%';
        $query->whereIn('id', function ($query) use ($term) {
            $query->select('id')
                ->from(function ($query) use ($term) {
                    $query->select('users.id')
                        ->from('users')
                        ->whereRaw("regexp_replace(users.first_name, '[^A-Za-z0-9]', '') ilike ?", [$term])
                        ->orWhereRaw("regexp_replace(users.last_name, '[^A-Za-z0-9]', '') ilike ?", [$term])
                        ->union(
                            $query->newQuery()
                                ->select('users.id')
                                ->from('users')
                                ->join('companies', 'users.company_id', '=', 'companies.id')
                                ->whereRaw("regexp_replace(companies.name, '[^A-Za-z0-9]', '') ilike ?", [$term])
                        );
                }, 'matches');
        });
    });
}
```