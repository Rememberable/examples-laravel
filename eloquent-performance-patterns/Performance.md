# Performance

## Run Authorization Policies in the Database
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->boolean('is_owner')->default(false);
    $table->timestamps();
});
Schema::create('customers', function (Blueprint $table) {
    $table->id();
    $table->foreignId('sales_rep_id')->constrained('users');
    $table->string('name');
    $table->string('city');
    $table->string('state');
    $table->timestamps();
});
// User model
public function customer()
{
    return $this->hasMany(Customer::class, 'sales_rep_id');
}
// Customer model
public function salesRep()
{
    return $this->belongsTo(User::class);
}
public function scopeVisibleTo($query, User $user)
{
    if (! $user->is_owner) {
        $query->where('sales_rep_id', $user->id);
    }
}
// controller
public function index()
{
    Auth::login(User::where('name', 'Sarah Seller')->first());

    $customers = Customer::query()
        ->visibleTo(Auth::user())
        ->with('salesRep')
        ->orderBy('name')
        ->paginate();

    return view('customers', ['customers' => $customers]);
}
```

## Faster Ordering With Compound Indexes
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('first_name');
    $table->string('last_name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
    $table->index(['last_name', 'first_name']);
});
// controller
public function index()
{
    $users = User::query()
        ->orderBy('last_name')
        ->orderBy('first_name')
        ->paginate();

    return view('users', ['users' => $users]);
}
```

## Options for Ordering by a HasOne Relationship
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('companies', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained('users');
    $table->string('name')->index();
    $table->timestamps();
});
// User model
public function company()
{
    return $this->hasOne(Company::class);
}
// controller
public function index()
{
    $users = User::query()
        ->select('users.*')
        ->join('companies', 'companies.user_id', '=', 'users.id')
        ->orderBy('companies.name')
        ->with('company')
        ->paginate();

    return view('users', ['users' => $users]);
}
```

## Options for Ordering by a BelongsTo Relationship
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->foreignId('company_id')->constrained('companies');
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('companies', function (Blueprint $table) {
    $table->id();
    $table->string('name')->index();
    $table->timestamps();
});
// User model
public function company()
{
    return $this->belongsTo(Company::class);
}
// Company model
public function users()
{
    return $this->hasMany(User::class);
}
// controller
public function index()
{
    $users = User::query()
        ->select('users.*')
        ->join('companies', 'companies.id', '=', 'users.company_id')
        ->orderBy('companies.name')
        ->with('company')
        ->paginate();

    return view('users', ['users' => $users]);
}
```

## Options for Ordering by a HasMany Relationship
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('logins', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained('users');
    $table->string('ip_address', 50);
    $table->timestamp('created_at');
});
// User model
public function logins()
{
    return $this->hasMany(Login::class);
}

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

public function scopeOrderByLastLogin($query)
{
    $query->orderByDesc(Login::select('created_at')
        ->whereColumn('user_id', 'users.id')
        ->latest()
        ->take(1)
    );
}
// Login model
class Login extends Model
{
    const UPDATED_AT = null;
}
// controller
public function index()
{
    $users = User::query()
        ->orderByLastLogin()
        ->withLastLogin()
        ->paginate();

    return view('users', ['users' => $users]);
}
```

## Options for Ordering by a BelongsToMany Relationship
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('town')->nullable();
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('books', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('author');
    $table->timestamps();
});
Schema::create('checkouts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained('users');
    $table->foreignId('book_id')->constrained('books');
    $table->date('borrowed_date');
});
// User model
public function books()
{
    return $this->belongsToMany(Book::class, 'checkouts')
        ->using(Checkout::class)
        ->withPivot('borrowed_date');
}
// Checkout model
protected $table = 'checkouts';

protected $casts = [
    'borrowed_date' => 'date',
];

public function user()
{
    return $this->belongsTo(User::class);
}
// Book model
public function user()
{
    return $this->belongsToMany(User::class, 'checkouts')
        ->using(Checkout::class)
        ->withPivot('borrowed_date');
}

public function lastCheckout()
{
    return $this->belongsTo(Checkout::class);
}

public function scopeWithLastCheckout($query)
{
    $query->addSelect(['last_checkout_id' => Checkout::select('checkouts.id')
        ->whereColumn('book_id', 'books.id')
        ->latest('borrowed_date')
        ->limit(1),
    ])->with('lastCheckout');
}
// controller
public function index()
{
    $books = Book::query()
        ->orderBy(User::select('name')
            ->join('checkouts', 'checkouts.user_id', '=', 'users.id')
            ->whereColumn('checkouts.book_id', 'books.id')
            ->latest('checkouts.borrowed_date')
            ->take(1)
        )
        ->withLastCheckout()
        ->with('lastCheckout.user')
        ->paginate();

    return view('books', ['books' => $books]);
}
```

## Ordering With NULLs Always Last
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('town')->nullable();
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('books', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->nullable()->constrained('users');
    $table->string('name');
    $table->string('author');
    $table->timestamps();
});
// Book model
public function user()
{
    return $this->belongsTo(User::class);
}
// User controller
public function index()
{
    $users = User::query()
        ->when(request('sort') === 'town', function ($query) {
            if (config('database.default') === 'mysql' || config('database.default') === 'sqlite') {
                $query->orderByRaw('town is null')
                    ->orderBy('town', request('direction'));
            }

            if (config('database.default') === 'pgsql') {
                $query->orderByNullsLast('town', request('direction'));
            }
        })
        ->orderBy('name')
        ->paginate();

    return view('users', ['users' => $users]);
}
// BooksController controller
public function index()
{
    $books = Book::query()
        ->with('user')
        ->orderByRaw('user_id is null')
        ->orderBy('name')
        ->paginate();

    return view('books', ['books' => $books]);
}
// AppServiceProvider 
public function boot()
{
    Paginator::defaultView('pagination');

    Builder::macro('orderByNullsLast', function ($column, $direction = 'asc') {
        $column = $this->getGrammar()->wrap($column);
        $direction = strtolower($direction) === 'asc' ? 'asc' : 'desc';

        return $this->orderByRaw("$column $direction nulls last");
    });
}
```

## Ordering By Custom Algorithms
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('features', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->string('title', 100);
    $table->string('status', 10);
    $table->timestamps();
    $table->rawIndex("(
        case
            when status = 'Requested' then 1
            when status = 'Approved' then 2
            when status = 'Completed' then 3
        end
    )", 'features_status_ranking_index');
});
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->foreignId('feature_id')->constrained('features');
    $table->foreignId('user_id')->constrained('users');
    $table->text('comment');
    $table->timestamps();
});
Schema::create('votes', function (Blueprint $table) {
    $table->bigIncrements('id');
    $table->foreignId('feature_id')->constrained('features');
    $table->foreignId('user_id')->constrained('users');
    $table->timestamps();
});
// Feature model
public function comments()
{
    return $this->hasMany(Comment::class);
}
public function votes()
{
    return $this->hasMany(Vote::class);
}
public function scopeOrderByStatus($query, $direction)
{
    $query->orderBy(DB::raw("
        case
            when status = 'Requested' then 1
            when status = 'Approved' then 2
            when status = 'Completed' then 3
        end
    "), $direction);
}
public function scopeOrderByActivity($query, $direction)
{
    if (config('database.default') === 'mysql' || config('database.default') === 'sqlite') {
        $query->orderBy(
            DB::raw('-(votes_count + (comments_count * 2))'),
            $direction
        );
    }

    if (config('database.default') === 'pgsql') {
        $votes = Vote::selectRaw('count(*)')
            ->whereColumn('votes.feature_id', 'features.id')
            ->toSql();

        $comments = Comment::selectRaw('count(*)')
            ->whereColumn('comments.feature_id', 'features.id')
            ->toSql();

        $query->orderBy(
            DB::raw("($votes) + (($comments) * 2)"),
            $direction
        );
    }
}
// controller
public function index()
{
    $features = Feature::query()
        ->withCount('comments', 'votes')
        ->when(request('sort'), function ($query, $sort) {
            switch ($sort) {
                case 'title': return $query->orderBy('title', request('direction'));
                case 'status': return $query->orderByStatus(request('direction'));
                case 'activity': return $query->orderByActivity(request('direction'));
            }
        })
        ->latest()
        ->paginate();

    return view('features', ['features' => $features]);
}
```

## Filtering and Sorting Anniversary Dates
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->date('birth_date')->nullable();
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();

    if (config('database.default') === 'mysql') {
        $table->rawIndex("(date_format(birth_date, '%m-%d')), name", 'users_birthday_name_index');
    }

    if (config('database.default') === 'sqlite') {
        $table->rawIndex("(strftime('%m-%d', birth_date)), name", 'users_birthday_name_index');
    }

    if (config('database.default') === 'pgsql') {
        DB::unprepared('
            create or replace function to_birthday(date timestamp)
            returns text language sql immutable as
            $f$ select to_char($1, \'MM-DD\') $f$
        ');

        $table->rawIndex('to_birthday(birth_date), name', 'users_birthday_name_index');
    }
});
// User model
public function scopeOrderByBirthday($query)
{
    if (config('database.default') === 'mysql') {
        $query->orderByRaw('date_format(birth_date, "%m-%d")');
    }

    if (config('database.default') === 'sqlite') {
        $query->orderByRaw('strftime("%m-%d", birth_date)');
    }

    if (config('database.default') === 'pgsql') {
        $query->orderByRaw('to_birthday(birth_date)');
    }
}

public function scopeOrderByUpcomingBirthdays($query)
{
    if (config('database.default') === 'mysql') {
        $query->orderByRaw('
            case
                when (birth_date + interval (year(?) - year(birth_date)) year) >= ?
                then (birth_date + interval (year(?) - year(birth_date)) year)
                else (birth_date + interval (year(?) - year(birth_date)) + 1 year)
            end
        ', [
            array_fill(0, 4, Carbon::now()->startOfWeek()->toDateString()),
        ]);
    }

    if (config('database.default') === 'sqlite') {
        throw new \Exception('This scope does not support SQLite.');
    }

    if (config('database.default') === 'pgsql') {
        throw new \Exception('This scope does not support Postgres.');
    }
}

public function scopeWhereBirthdayThisWeek($query)
{
    $dates = Carbon::now()->startOfWeek()
        ->daysUntil(Carbon::now()->endOfWeek())
        ->map(fn ($date) => $date->format('m-d'));

    if (config('database.default') === 'mysql') {
        $query->whereRaw('date_format(birth_date, "%m-%d") in (?,?,?,?,?,?,?)', iterator_to_array($dates));
    }

    if (config('database.default') === 'sqlite') {
        $query->whereRaw('strftime("%m-%d", birth_date) in (?,?,?,?,?,?,?)', iterator_to_array($dates));
    }

    if (config('database.default') === 'pgsql') {
        $query->whereRaw('to_birthday(birth_date) in (?,?,?,?,?,?,?)', iterator_to_array($dates));
    }
}
// controller
public function index()
{
    $users = User::query()
        ->whereBirthdayThisWeek()
        ->orderByBirthday()
        // ->orderByUpcomingBirthdays()
        ->orderBy('name')
        ->paginate();

    return view('users', ['users' => $users]);
}
```

## Filtering and Sorting Anniversary Dates
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('devices', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('brand');
    $table->string('resolution');
    $table->timestamps();
});
public function up()
{
    if (config('database.default') === 'mysql') {
        // https://www.drupal.org/project/natsort
        DB::unprepared("
            drop function if exists naturalsort;
            create function naturalsort(s varchar(255)) returns varchar(255)
                no sql
                deterministic
            begin
                declare orig   varchar(255)  default s;
                declare ret    varchar(255)  default '';
                if s is null then
                    return null;
                elseif not s regexp '[0-9]' then
                    set ret = s;
                else
                    set s = replace(replace(replace(replace(replace(s, '0', '#'), '1', '#'), '2', '#'), '3', '#'), '4', '#');
                    set s = replace(replace(replace(replace(replace(s, '5', '#'), '6', '#'), '7', '#'), '8', '#'), '9', '#');
                    set s = replace(s, '.#', '##');
                    set s = replace(s, '#,#', '###');
                    begin
                        declare numpos int;
                        declare numlen int;
                        declare numstr varchar(255);
                        lp1: loop
                            set numpos = locate('#', s);
                            if numpos = 0 then
                                set ret = concat(ret, s);
                                leave lp1;
                            end if;
                            set ret = concat(ret, substring(s, 1, numpos - 1));
                            set s    = substring(s,    numpos);
                            set orig = substring(orig, numpos);
                            set numlen = char_length(s) - char_length(trim(leading '#' from s));
                            set numstr = cast(replace(substring(orig,1,numlen), ',', '') as decimal(13,3));
                            set numstr = lpad(numstr, 15, '0');
                            set ret = concat(ret, '[', numstr, ']');
                            set s    = substring(s,    numlen+1);
                            set orig = substring(orig, numlen+1);
                        end loop;
                    end;
                end if;
                set ret = replace(replace(replace(replace(replace(replace(replace(ret, ' ', ''), ',', ''), ':', ''), '.', ''), ';', ''), '(', ''), ')', '');
                return ret;
            end;
        ");
    }

    if (config('database.default') === 'sqlite') {
        throw new \Exception('This lesson does not support SQLite.');
    }

    if (config('database.default') === 'pgsql') {
        // http://www.rhodiumtoad.org.uk/junk/naturalsort.sql
        DB::unprepared('
          create or replace function naturalsort(text)
          returns bytea language sql immutable strict as
          $f$ select string_agg(convert_to(coalesce(r[2],length(length(r[1])::text) || length(r[1])::text || r[1]),\'SQL_ASCII\'),\'\x00\')
          from regexp_matches($1, \'0*([0-9]+)|([^0-9]+)\', \'g\') r; $f$;
      ');
    }
}

/**
 * Reverse the migrations.
 *
 * @return void
 */
public function down()
{
    if (config('database.default') === 'mysql') {
        DB::unprepared('drop function if exists naturalsort');
    }

    if (config('database.default') === 'sqlite') {
        throw new \Exception('This lesson does not support SQLite.');
    }

    if (config('database.default') === 'pgsql') {
        DB::unprepared('drop function if exists naturalsort');
    }
}
// controller
public function index()
{
    $devices = Device::query()
        ->orderByRaw('naturalsort(name)')
        ->paginate();

    return view('devices', ['devices' => $devices]);
}
```

## Full Text Searching With Rankings
```php
<?php
// migrations
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->foreignId('author_id')->constrained('users');
    $table->string('title');
    $table->string('slug');
    $table->longText('body');
    $table->dateTime('published_at');
    $table->timestamps();
});

if (config('database.default') === 'mysql') {
    DB::statement('CREATE FULLTEXT INDEX posts_fulltext_index ON posts(title, body) WITH PARSER ngram');
}

if (config('database.default') === 'sqlite') {
    throw new \Exception('This lesson does not support SQLite.');
}

if (config('database.default') === 'pgsql') {
    DB::statement('ALTER TABLE posts ADD COLUMN searchable TSVECTOR');
    DB::statement('CREATE INDEX posts_searchable_gin ON posts USING GIN(searchable)');
    DB::statement("CREATE TRIGGER posts_searchable_update BEFORE INSERT OR UPDATE ON posts FOR EACH ROW EXECUTE PROCEDURE tsvector_update_trigger('searchable', 'pg_catalog.english', 'title', 'body')");
}
// User model
public function posts()
{
    return $this->hasMany(Post::class, 'author_id');
}
// Post model
protected $casts = [
    'published_at' => 'datetime',
];

public function author()
{
    return $this->belongsTo(User::class, 'author_id');
}
// controller
public function index()
{
    $posts = Post::query()
        ->with('author')
        ->when(request('search'), function ($query, $search) {
            if (config('database.default') === 'mysql') {
                $query->whereRaw('match(title, body) against(? in boolean mode)', [$search])
                    ->selectRaw('*, match(title, body) against(? in boolean mode) as score', [$search]);
            }

            if (config('database.default') === 'sqlite') {
                throw new \Exception('This lesson does not support SQLite.');
            }

            if (config('database.default') === 'pgsql') {
                $search = implode(' | ', array_filter(explode(' ', $search)));
                $query->selectRaw("*, ts_rank(searchable, to_tsquery('english', ?)) as score", [$search])
                    ->whereRaw("searchable @@ to_tsquery('english', ?)", [$search])
                    ->orderByRaw("ts_rank(searchable, to_tsquery('english', ?)) desc", [$search]);
            }
        }, function ($query) {
            $query->latest('published_at');
        })
        ->paginate();

    return view('posts', ['posts' => $posts]);
}
```