---
author: Siddharth Srinivasan
pubDatetime: 2024-05-18T15:22:00Z
title: The Laravel Mindset
slug: the-laravel-mindset
featured: true
tags:
  - summary
description: Getting into the Laravel developer mindset by buidling a full stack app using Laravel and Inertia.
---

## Table of contents

## Mail

HELO - paid

We can keep the `MAIL_MAILER` value as `log` in our `.env` if we would like to catch all emails in the local storage folder.

[Mailpit](https://github.com/axllent/mailpit) is a free open-source alternative.

## Defining our data

Create a `Post` model:

```
php artisan make:model
```

Create the model along with `Factory`, `Migration` and `Resource Controller` options.

Update the `posts` table migration to include columns for `title` and `body` values.

```
$table->string('title');
$table->longText('body');
```

We use `longText` column type for the `body` as the `string` type has a restriction of a maximum of 255 characters.

An opinionated code organization convention is to group the columns definitions in the following order: `ids`, `standard columns`, `date columns`.

Let's add the foreign id column to the `users` table as any post would have an author who in turn is a user.

```
$table->foreignIdFor(User::class)->constrained()->restrictOnDelete();
```

The `restrictOnDelete` method makes sure that the table constrains are handled by our application logic rather than by the database. The database would just throw an exception whenever we try to delete a row that has constrains on another table.

Let's create the `Comment` model.

We do not need the `Resource Controller` option in this case as we would have no use-case for indexing the comments table; i.e, a comment is always going to be associated with a user and a post. There would never be a case where we might have to list all the comments without user/post constrains.

Let's update the migration `up` method for the `comments` table.

```
$table->foreignIdFor(User::class)->constrained()->restrictOnDelete();
$table->foreignIdFor(Post::class)->constrained()->cascadeOnDelete();
$table->longText('body');
```

In the case of the `Post` relation, we add the `cascadeOnDelete` method as we want to delete all `comments` on a post when the related `post` is deleted.

Now, let's update the relationships on the three models we've just created in their model classes.

A user `has many` posts and also `has many` comments. So, let's update `app/Models/User.php`

```php
public function posts(): HasMany
{
    return $this->hasMany(Post::class);
}

public function comments(): HasMany
{
    return $this->hasMany(Comment::class);
}
```

Let's add the inverse relations in the `Post` and `Comment` models.

A post `belongs to` a user and `has many` comments.

```php
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function comments(): HasMany
{
    return $this->hasMany(Comment::class);
}
```

A comment `belongs to` a user and a post.

```php
public function user(): BelongsTo
{
    $this->belongsTo(User::class);
}

public function post(): BelongsTo
{
    $this->belongsTo(Post::class);
}
```

## Seeding our database

Let's seed data using Laravel's model factories.

These data created using model factories are not only useful for using the app during local development, they are also useful for creating data on the fly for tests.

Note: `User` factory comes updated on a new Laravel project.

Let's start with the `Post` factory.

```php
public function definition(): array
{
    return [
        'user_id' => User::factory(),
        'title' => str(fake()->sentence())->beforeLast('.')->title(),
        'body' => fake()->realText(600),
    ];
}
```

Notice that we have used `User::factory()` rather than `User::factory()->create()` for the `user_id` because we do not want to always create a new user when we are creating a post data using the Post model factory. We want to create a new user only when we do not have one that we can use. Laravel intelligently creates a new user when needed using the User model factory. This is how we define foreign relations in Laravel model factories.

For the "title" column, we have used a sentence type and since a sentence would have a `.` at the end, we use the `beforeLast` helper to return only the text preceding the `.`; We then use the `title` helper to make each word in the "title" capitalized.

For the "body" column, we could have used `fake()->paragraph()` helper but that would only generate "Lorem ipsum..." text. We want to generate meaningful english text and so we use the `realText` helper which would generate passages from real english books.

We next update the `Comment` factory.

```php
public function definition(): array
{
    return [
        'user_id' => User::factory(),
        'post_id' => Post::factory(),
        'body' => fake()->realText(250),
    ];
}
```

Now that the model factories are updated, let's update the database seeder.

Let's create 10 new users.

```php
$users = User::factory(10)->create();
```

Let's also create a known user so that we can easily login whenever we need to.

```php
$sid = User::factory()->create([
    'name' => 'Siddharth Srinivasan',
    'email' => 'test@example.com',
]);
```

Now, it's time to run the seeder.

```
php artisan db:seed
```

Now we cannot seed the database using the above command once again because, it would try to create another user with the email `test@example.com` and would result in a database constraint.

So, if we want to seed the database again, we do it along with running a fresh migration.

```
php artisan migrate:fresh --seed
```

Let's update our database seeder so that each user that is added has 20 posts created by them.

```php
$users = User::factory(10)
    ->has(Post::factory(20))
    ->create();
```

Let's seed our posts table to have 200 posts.

```php
Post::factory(200)->create();
```

The problem with the above approach is that this would create 200 new users (1 for each new post). What if there is a way to reuse, ...or recycle... the existing users that we have already created above. Yes, there is a way.

```php
// Let's remove the has method.
$users = User::factory(10);

$posts = Post::factory(200)->recycle($users)->create();
```

Let's do the same for comments.

```php
$comments = Comment::factory(100)
    ->recycle($users)
    ->recycle($posts)
    ->create();
```

Let's seed posts and comments data for the one known user that we created above.

```php
$sid = User::factory()
    ->has(Post::factory(45))
    ->has(Comment::factory(100)->recycle($posts))
    ->create([
    'name' => 'Siddharth Srinivasan',
    'email' => 'test@example.com',
]);
```

## Running our tests

Let's run our existing tests that come with a new Laravel project.

```
php artisan test
```

Sure enough, all the tests pass (if Pest is setup correctly)

At this point, all the data that we seeded in the database should be gone. This is because our test suite uses the same database as our application.

Let's fix that. In the `phpunit.xml` file, we can remove the property row for `DB_CONNECTION` as the default db connection is MySQL and that is what we want for tests as well. Next up, let's update the value for `DB_DATABASE` to `our_app_database_test` - this is only a convention, assuming the app uses the database with name `our_app_database`. Now if we run our tests, it would not affect our primary app database.

Tip: We can run tests in parallel using `php artisan test -p`. This would spawn up multiple databases for tests to make them run in parallel and complete faster.

Tip: In the `Pest.php` file, we can replace `RefreshDatabase` trait to `LazilyRefreshDatabase` so that the database migrations and related actions run only if the test touches the database. This would help to run the tests faster.

## Creating an index of Posts using TDD
