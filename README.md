# Keep track of outgoing emails and associate sent emails with Eloquent models

[![Latest Version on Packagist](https://img.shields.io/packagist/v/wnx/laravel-sends.svg?style=flat-square)](https://packagist.org/packages/wnx/laravel-sends)
[![run-tests](https://github.com/stefanzweifel/laravel-sends/actions/workflows/run-tests.yml/badge.svg)](https://github.com/stefanzweifel/laravel-sends/actions/workflows/run-tests.yml)
[![Check & fix styling](https://github.com/stefanzweifel/laravel-sends/actions/workflows/php-cs-fixer.yml/badge.svg)](https://github.com/stefanzweifel/laravel-sends/actions/workflows/php-cs-fixer.yml)
[![Total Downloads](https://img.shields.io/packagist/dt/wnx/laravel-sends.svg?style=flat-square)](https://packagist.org/packages/wnx/laravel-sends)

This package helps you to keep track of outgoing emails in your Laravel application. In addition, you can associate Models to the sent out emails.

Here are a few short examples what you can do:

```php
// An email is sent to $user. The passed $product and $user models will be 
// associated with the sent out email generated by the `ProductReviewMail` mailable.
Mail::to($user)->send(new ProductReviewMail($product, $user));
```

In your application, you can then fetch all sent out emails for the user or the product.

```php
$user->sends()->get();
$product->sends()->get();
```

Or you can fetch all sent out emails for the given Mailable class.

```php
Send::byMailClass(ProductReviewMail::class)->get();
```

Each `Send`-model holds the following information:

- FQN of mailable class
- subject
- from address
- reply to address
- to address
- cc adresses
- bcc adresses

Additionally, the `sends`-table has the following columns which can be filled by your own application ([learn more](https://github.com/stefanzweifel/laravel-sends#further-usage-of-the-sends-table)).

- `delivered_at`
- `last_opened_at`
- `opens`
- `clicks`
- `last_clicked_at`
- `complained_at`
- `bounced_at`
- `permanent_bounced_at`
- `rejected_at`

## Installation

You can install the package via composer:

```bash
composer require wnx/laravel-sends
```

Then, publish and run the migrations:

```bash
php artisan vendor:publish --tag="sends-migrations"
php artisan migrate
```

Optionally, you can publish the config file with:

```bash
php artisan vendor:publish --tag="sends-config"
```

This is the contents of the published config file:

```php
return [
    /*
     * The fully qualified class name of the send model.
     */
    'send_model' => \Wnx\Sends\Models\Send::class,

    'headers' => [
        /**
         * Header containing unique ID of the sent out mailable class.
         */
        'custom_message_id' => env('SENDS_HEADERS_CUSTOM_MESSAGE_ID', 'X-Laravel-Message-ID'),

        /**
         * Header containing the encrypted FQN of the mailable class.
         */
        'mail_class' => env('SENDS_HEADERS_MAIL_CLASS', 'X-Laravel-Mail-Class'),

        /**
         * Header containing an encrypted JSON object with information which
         * Eloquent models are associated with the mailable class.
         */
        'models' => env('SENDS_HEADERS_MAIL_MODELS', 'X-Laravel-Mail-Models'),
    ],
];
```

## Usage

After the installation is done, update your applications `EventServiceProvider` to listen to the `MessageSent` event. Add the `StoreOutgoingMailListener`-class as a listener.

```php
// app/Providers/EventServiceProvider.php
protected $listen = [
    // ...
    \Illuminate\Mail\Events\MessageSent::class => [
        \Wnx\Sends\Listeners\StoreOutgoingMailListener::class,
    ],
]
```

The metadata of **all** outgoing emails created by mailables or notifications is now being stored in the `sends`-table. (Note that you currently can only associate models to mailables but not to notifiations)

Read further to learn how to store the name and how to associate models with a Mailable class.

### Store Mailable class name on Send Model

By default the Event Listener stores the mails subject and the recipient adresses. That's nice, but we can do better.
It can be beneficial for your application to know which Mailable class triggered the sent email. 

To store this information, add the `StoreMailables`-trait to your Mailable classes like below.
Then call the `storeClassName()`-method inside the `build`-method.

```php
class ProductReviewMail extends Mailable
{
    use SerializesModels;
    use StoreMailables;

    public function __construct(public User $user, public Product $product)
    {
    }

    public function build()
    {
        return $this
            ->storeClassName()
            ->view('emails.products.review')
            ->subject("$this->product->name waits for your review");
    }
}
```

The method will add a `X-Laravel-Mail-Class`-header to the outgoing email containing the fully qualified name (FQN) of the Mailable class as an encrypted string. (eg. `App\Mails\ProductReviewMail`)

The package's event listener will then look for the header, decrypt the value and store it in the database.

### Associate Sends with Related Models

Now that you already keep track of all outgoing emails, wouldn't it be great if you could access the records from an associated Model?
In the example above we send a `ProductReviewMail` to a user. Wouldn't it be great to see how many emails you have sent for a given `Product`-model?

To achieve this, you have to add the `HasSends`-interface and the `HasSendsTrait` trait to your models.

```php
use Wnx\Sends\Contracts\HasSends;
use Wnx\Sends\Support\HasSendsTrait;

class Product extends Model implements HasSends
{
    use HasSendsTrait;
}
```

You can now call the `associateWith()`-method within the `build()`-method and pass it an array of Models you want to associate with the Mailable class.

```php
class ProductReviewMail extends Mailable
{
    use SerializesModels;
    use StoreMailables;

    public function __construct(private User $user, private Product $product)
    {
    }

    public function build()
    {
        return $this
            ->storeClassName()
            ->associateWith([$this->product])
            ->view('emails.products.review')
            ->subject("$this->user->name, $this->product->name waits for your review");
    }
}
```

You can now access the sent out emails from the product's `send`-relationship.

```php
$product->sends()->get();
```

#### Automatically associate Models with Mailables

If you do not pass an argument to the `associateWith`-method, the package will automatically associate all public properties which implement the `HasSends`-interface with the Mailable class.

In the example below, the `ProductReviewMail`-Mailable will automatically be associated with the `$product` Model, as `Product` implements the `HasSends` interface. The `$user` model will be ignored, as it's declared as a private property.

```php
class ProductReviewMail extends Mailable
{
    use SerializesModels;
    use StoreMailables;

    public function __construct(private User $user, public Product $product)
    {
    }

    public function build()
    {
        return $this
            ->associateWith()
            ->view('emails.products.review')
            ->subject("$this->user->name, $this->product->name waits for your review");
    }
}
```

### Attach custom Message ID to Mails

If you're sending emails through AWS SES or a similar service, you might want to identify the sent email in the future (for example when a webhook for the "Delivered"-event is sent to your application).

The package comes with an event listener helping you here. Update the EventServiceProvider to listen to the `MessageSending` event and add the `AttachCustomMessageIdListener` as a listener. 
A `X-Laravel-Message-ID` will be attached to all outgoing emails.

```php
// app/Providers/EventServiceProvider.php
protected $listen = [
    // ...
    \Illuminate\Mail\Events\MessageSending::class => [
        \Wnx\Sends\Listeners\AttachCustomMessageIdListener::class,
    ],
]
```

### Prune Send Models

By default, `Send`-models are kept forever in your database. If your application sends thousands of emails per day, you might want to prune records after a couple of days or months.

To do that, use the [prunable](https://laravel.com/docs/master/eloquent#pruning-models) feature of Laravel.

Create a new `Send`-model in your `app/Models` that extends `Wnx\Sends\Models\Send`.
Then add the `Prunable`-trait and set up the `prunable()`-method to your liking. The example below deletes all `Send`-models older than 1 month.

```php
// app/Models/Send.php
namespace App\Models;

use Illuminate\Database\Eloquent\Prunable;
use Wnx\Sends\Models\Send as BaseSend;

class Send extends BaseSend
{
    use Prunable;

    public function prunable()
    {
        return static::where('created_at', '<=', now()->subMonth());
    }
}
```

Optionally you can also update the configuration, so that the package internally uses your Send model.

```php
// config/sends.php
/*
 * The fully qualified class name of the send model.
 */
'send_model' => \App\Models\Send::class,
```

## Further Usage of the `sends`-table

As you might have noticed, the `sends`-table comes with more columns than that are currently filled by the package. This is by design.

You are encouraged to write your own application logic to fill these currently empty columns. For example, if you're sending emails through AWS SES, I highly encourage you to use the [renoki-co/laravel-aws-webhooks](https://github.com/renoki-co/laravel-aws-webhooks) package to handle AWS SNS webhooks.

A controller that handles the "Delivered" event might look like this.

```php
class AwsSnsSesWebhookController extends SesWebhook {
    protected function onDelivery(array $message)
    {
        $messageIdHeader = collect(Arr::get($message, 'mail.headers', []))
            ->where('name', 'X-Laravel-Message-ID')
            ->first();

        if ($messageIdHeader === null) {
            return;
        }

        $send = Send::where('message_id', $messageIdHeader['value'])->first();

        if ($send === null) {
            return;
        }

        $send->delivered_at = now();
        $send->save();
    }
}
```

## Testing

```bash
composer test
```

## Changelog

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.

## Contributing

Please see [CONTRIBUTING](.github/CONTRIBUTING.md) for details.

## Security Vulnerabilities

Please review [our security policy](../../security/policy) on how to report security vulnerabilities.

## Credits

- [Stefan Zweifel](https://github.com/stefanzweifel)
- [All Contributors](../../contributors)

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
