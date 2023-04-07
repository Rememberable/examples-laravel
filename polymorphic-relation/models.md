# Polymorphic relation. Models

## Structure of models folder (All classes in own folder)

```php
<?php

class User extends Model
{
    protected $fillable = ['name', 'email', 'password'];
}

class Favorite extends Model
{
    protected $fillable = ['user_id'];
}

class Post extends Model
{
    use Favoritable;

    public function comments()
    {
        return $this->hasMany(Comment::class);
    }
}

class Comment extends Model
{
    use Favoritable;

    public function post()
    {
        return $this->belongsTo(Post::class);
    }
}

trait Favoritable
{
    public function favorites()
    {
        return $this->morphMany(Favorite::class, 'favoritable');
    }

    public function scopeFavoritedBy($query, User $user)
    {
        return $query->whereHas('favorites', function ($query) use ($user) {
            $query->where('user_id', $user->id);
        });
    }

    public function isFavoritedBy(User $user)
    {
        return $this->favorites()
            ->where('user_id', $user->id)
            ->exists();
    }

    public function favorite()
    {
        $this->favorites()->save(
            new Favorite(['user_id' => auth()->id()])
        );
    }
}
```