---
title: Using Laravel Model Factories in PHPUnit Data Providers
description: Using Laravel model factories inside PHPUnit data providers isn't as straightforward as you would think. This article shows you how can pull this off.
tags: PHP, PHPUnit, Laravel
---

If you've ever tried to use a [Laravel model factory](https://laravel.com/docs/10.x/eloquent-factories) inside a [PHPUnit data provider](https://docs.phpunit.de/en/11.0/writing-tests-for-phpunit.html#data-providers), then you were likely greeted with an error message resembling `"A facade root has not been set"`. After scratching your head for a few minutes, you probably (wisely) decided that life is too short and then promptly scrapped the whole approach. The issue that you encountered is an expected consequence of the lifecycle of PHPUnit tests. In PHPUnit, data providers are called before a test's `setUp` method. If you look inside the `Illuminate\Foundation\Testing\TestCase` class, which is the abstract class that Laravel feature tests ultimately extend, you'll see the following snippet:

{% highlight php %}
protected function setUp(): void
{
    static::$latestResponse = null;

    Facade::clearResolvedInstances();

    if (! $this->app) {
        $this->refreshApplication();

        ParallelTesting::callSetUpTestCaseCallbacks($this);
    }

    $this->setUpTraits();

    foreach ($this->afterApplicationCreatedCallbacks as $callback) {
        $callback();
    }

    Model::setEventDispatcher($this->app['events']);

    $this->setUpHasRun = true;
}
{% endhighlight %}

The `setUp` method inside that class is responsible for booting the Laravel framework in tests via the `refreshApplication` method. As data providers are executing before that `setUp` method has been called, you're not going to have access to Laravel functionality such as facades and model factories at that point. You might think that this is the end of the road for using model factories inside your data providers. Fear not, there's a neat trick that you can use to get around this limitation. If you wrap the provider's data set within a closure, this will cause the expression to be evaluated lazily.

{% highlight php %}
<?php

namespace Tests\Feature;

use App\Models\User;
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\Attributes\Test;
use Tests\TestCase;
use Closure;
use Generator;

class UserTest extends TestCase
{
    #[Test]
    #[DataProvider('singularUserProvider')]
    public function testing_a_single_user_model(Closure $userClosure): void
    {
        $user = $userClosure();

        $this->assertInstanceOf(User::class, $user);
        $this->assertSame('Greg Mayes', $user->name);
    }

    #[Test]
    #[DataProvider('multipleUsersProvider')]
    public function testing_multiple_user_models(Closure $usersClosure): void
    {
        foreach ($usersClosure() as $user) {
            $this->assertInstanceOf(User::class, $user);
            $this->assertNotEmpty($user->name);
        }
    }

    public static function singularUserProvider(): Generator
    {
        yield [static fn (): User => User::factory()->make(['name' => 'Greg Mayes'])];
    }

    public static function multipleUsersProvider(): Generator
    {
        yield [
            static fn (): array => [
                User::factory()->make(['name' => 'Greg Mayes']),
                User::factory()->make(['name' => 'Joe Bloggs']),
            ]
        ];
    }
}
{% endhighlight %}

The above approach does have the downside of requiring you to invoke the `Closure` inside the body of your test, but it's a relatively small price to pay. For the sake of completeness, I've included examples of returning a singular `User` model in a data set as well as returning an array of `User` models. 

If you want to learn more about PHPUnit data providers outside of the previously linked documentation, then I would highly recommend [this blog post](https://barlow.dev/posts/phpunits-data-providers-are-your-friend) by Matt Barlow.