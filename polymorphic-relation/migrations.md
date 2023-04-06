# Polymorphic relation. Migrations

## Structure of databases

<table>
<tr>
    <td> users </td> 
    <td> posts </td>
</tr>
<tr><td>

```php
<?php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('password');
    $table->rememberToken();
    $table->timestamps();
});
```

</td><td>

```php
<?php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('body');
    $table->timestamps();
});
```

</td></tr></table>

<table>
<tr>
    <td> comments </td>
    <td> favorites </td>
</tr>
<tr><td>

```php
<?php
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->integer('post_id')->index();
    $table->string('body');
    $table->timestamps();
});
```

</td><td>

```php
<?php
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->integer('user_id')->index();
    $table->integer('favoritable_id');
    $table->integer('favoritable_type');
    $table->timestamps();

    $table->unique(['user_id', 'favoritable_id', 'favoritable_type']);
});
```

</td></tr></table>