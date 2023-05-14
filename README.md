Change package name temporary to "miladtech/tmp-miladtech-validating" to publish on packagist.

This is temporary forked package until a new release published by author, fully supports Laravel v8+


Validating, a validation trait for Laravel
==========================================

Validating is a trait for Laravel Eloquent models which ensures that models meet their validation criteria before being saved. If they are not considered valid the model will not be saved and the validation errors will be made available.

Validating allows for multiple rulesets, injecting the model ID into `unique` validation rules and raising exceptions on failed validations. It's small and flexible to fit right into your workflow and help you save valid data only.

# Installation
Simply go to your project directory where the `composer.json` file is located and type:

```sh
composer require miladtech/validating
```

## Overview
First, add the trait to your model and add your validation rules and messages as needed.

```php
use MiladTech\Validating\ValidatingTrait;

class Post extends Eloquent
{
	use ValidatingTrait;

	protected $rules = [
		'title'   => 'required',
		'slug'    => 'required|unique:posts,slug',
		'content' => 'required'
	];
}
```

You can also add the trait to a `BaseModel` if you're using one and it will work on all models that extend from it, otherwise you can just extend `MiladTech\Validating\ValidatingModel` instead of `Eloquent`.

*Note: you will need to set the `$rules` property on any models that extend from a `BaseModel` that uses the trait, or otherwise set an empty array as the `$rules` for the `BaseModel`. If you do not, you will inevitably end up with `LogicException with message 'Relationship method must return an object of type Illuminate\Database\Eloquent\Relations\Relation'`.*

Now, you have access to some pleasant functionality.

```php
// Check whether the model is valid or not.
$post->isValid(); // true

// Or check if it is invalid or not.
$post->isInvalid(); // false

// Once you've determined the validity of the model,
// you can get the errors.
$post->getErrors(); // errors MessageBag
```

Model validation also becomes really simple.

```php
if ( ! $post->save()) {
    // Oops.
    return redirect()->route('posts.create')
        ->withErrors($post->getErrors())
        ->withInput();
}

return redirect()->route('posts.show', $post->id)
    ->withSuccess("Your post was saved successfully.");
```

Otherwise, if you prefer to use exceptions when validating models you can use the `saveOrFail()` method. Now, an exception will be raised when you attempt to save an invalid model.

```php
$post->saveOrFail();
```

You *don't need to catch the exception*, if you don't want to. Laravel knows how to handle a `ValidationException` and will automatically redirect back with form input and errors. If you want to handle it yourself though you may.

```php
try {
    $post->saveOrFail();

} catch (MiladTech\Validating\ValidationException $e) {
    $errors = $e->getErrors();

    return redirect()->route('posts.create')
        ->withErrors($errors)
        ->withInput();
}
```

Note that you can just pass the exception to the `withErrors()` method like `withErrors($e)` and Laravel will know how to handle it.

### Bypass validation
If you're using the model and you wish to perform a save that bypasses validation you can. This will return the same result as if you called `save()` on a model without the trait.

```php
$post->forceSave();
```

### Validation exceptions by default
If you would prefer to have exceptions thrown by default when using the `save()` method instead of having to use `saveOrFail()` you can just set the following property on your model or `BaseModel`.

```php
/**
 * Whether the model should throw a ValidationException if it
 * fails validation. If not set, it will default to false.
 *
 * @var boolean
 */
protected $throwValidationExceptions = true;
```

If you'd like to perform a one-off save using exceptions or return values, you can use the `saveOrFail()` and `saveOrReturn` methods.

### Validation messages
To show custom validation error messages, just add the `$validationMessages` property to your model.

```php
/**
 * Validation messages to be passed to the validator.
 *
 * @var array
 */
protected $validationMessages = [
    'slug.unique' => "Another post is using that slug already."
];
```

### Unique rules
You may have noticed we're using the `unique` rule on the slug, which wouldn't work if we were updating a persisted model. Luckily, Validation will take care of this for you and append the model's primary key to the rule so that the rule will work as expected; ignoring the current model.

You can adjust this functionality by setting the `$injectUniqueIdentifier` property on your model.

```php
/**
 * Whether the model should inject it's identifier to the unique
 * validation rules before attempting validation. If this property
 * is not set in the model it will default to true.
 *
 * @var boolean
 */
protected $injectUniqueIdentifier = true;
```

Out of the box, we support the Laravel provided `unique` rule. We also support the popular [felixkiss/uniquewith-validator](https://github.com/felixkiss/uniquewith-validator) rule, but you'll need to opt-in. Just add `use \MiladTech\Validating\Injectors\UniqueWithInjector` after you've imported the validating trait.

It's easy to support additional injection rules too, if you like. Say you wanted to support an additional rule you've got called `unique_ids` which simply takes the model's primary key (for whatever reason). You just need to add a camel-cased rule which accepts any existing parameters and the field name, and returns the replacement rule.

```php
/**
 * Prepare a unique_ids rule, adding a model identifier if required.
 *
 * @param  array  $parameters
 * @param  string $field
 * @return string
 */
protected function prepareUniqueIdsRule($parameters, $field)
{
    // Only perform a replacement if the model has been persisted.
    if ($this->exists) {
        return 'unique_ids:' . $this->getKey();
    }

    return 'unique_ids';
}
```

In this case if the model has been saved and has a primary key of `10`, the rule `unique_ids` will be replaced with `unique_ids:10`.

### Events
Various events are fired by the trait during the validation process which you can hook into to impact the validation process.

To hook in, you first need to add the `$observeables` property onto your model (or base model). This simply lets Eloquent know that your model can respond to these events.

```php
/**
 * User exposed observable events
 *
 * @var array
 */
protected $observables = ['validating', 'validated'];
```

When validation is about to occur, the `eloquent.validating: ModelName` event will be fired, where the `$event` parameter will be `saving` or `restoring`. For example, if you were updating a namespaced model `App\User` the event would be `eloquent.validating: App\User`. If you listen for any of these events and return a value you can prevent validation from occurring completely.

```php
Event::listen('eloquent.validating:*', function($model, $event) {
    // Pseudo-Russian roulette validation.
    if (rand(1, 6) === 1) {
        return false;
    }
});
```

After validation occurs, there are also a range of `validated` events you can hook into, for the `passed`, `failed` and `skipped` events. For the above example failing validation, you could get the event `eloquent.validated: App\User`.


## Controller usage

This example keeps your code clean by allowing the FormRequest to handle your form validation and the model to handle its own validation. By enabling validation exceptions you can reduce repetitive controller code (try/catch blocks) and handle model validation exceptions globally (your form requests should keep your models valid, so if your model becomes invalid it's an *exceptional* event).

```php
<?php namespace App\Http\Controllers;

use App\Http\Requests\PostFormRequest;
use Illuminate\Routing\Controller;

class PostsController extends Controller
{
    protected $post;

    public function __construct(Post $post)
    {
        $this->post = $post;
    }

    // ...

    public function store(PostFormRequest $request)
    {
        // Post will throw an exception if it is not valid.
        $post = $this->post->create($request->input());

        // Post was saved successfully.
        return redirect()->route('posts.show', $post);
    }
}
```

You can then catch a model validation exception in your `app/Exceptions/Handler.php` and deal with it as you need.

```php
public function render($request, Exception $e)
{
    if ($e instanceof \MiladTech\Validating\ValidationException) {
        return back()->withErrors($e)->withInput();
    }

    parent::render($request, $e);
}
```
