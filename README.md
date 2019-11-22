# { json:api } Client

[![PHP from Packagist](https://img.shields.io/packagist/php-v/swisnl/json-api-client.svg)](https://packagist.org/packages/swisnl/json-api-client)
[![Latest Version on Packagist](https://img.shields.io/packagist/v/swisnl/json-api-client.svg)](https://packagist.org/packages/swisnl/json-api-client)
[![Software License](https://img.shields.io/packagist/l/swisnl/json-api-client.svg)](LICENSE)
[![Build Status](https://travis-ci.org/swisnl/json-api-client.svg?branch=master)](https://travis-ci.org/swisnl/json-api-client)
[![Scrutinizer Coverage](https://img.shields.io/scrutinizer/coverage/g/swisnl/json-api-client.svg)](https://scrutinizer-ci.com/g/swisnl/json-api-client/?branch=master)
[![Scrutinizer Code Quality](https://img.shields.io/scrutinizer/g/swisnl/json-api-client.svg)](https://scrutinizer-ci.com/g/swisnl/json-api-client/?branch=master)
[![Made by PrimeXConnect](https://img.shields.io/badge/%F0%9F%9A%80-made%20by%20SWIS-%23D9021B.svg)](http://www.primexconnect.com/)

A PHP package for mapping remote [{json:api}](http://jsonapi.org/) resources to Eloquent like models and collections.

A copy of [json-api-client](https://github.com/swisnl/json-api-client/)

### @todo this documentation needs to be updated.

## Upgrade from 1.0.x to 1.2.x

The dependency `swisnl/json-api-client` has been upgraded from '0.10.x' to '0.20.x', supporting Lumen 5.8. In your `ServiceProvider`

- Replace `use Swis\JsonApi\Client\Interfaces\ParserInterface;
` with `use Swis\JsonApi\Client\Interfaces\ResponseParserInterface;` 
- Replace `use Swis\JsonApi\Client\ItemDocumentSerializer;` with `use PXC\JsonApi\Client\ItemDocumentSerializerInterface;`
- Change the other part of the file accordingly.

## Installation

``` bash
composer require pxc/json-api-client
```

N.B. Make sure you have installed a HTTP Client before you install this package or install one at the same time e.g. `composer require swisnl/json-api-client php-http/guzzle6-adapter`.

### HTTP Client

We are decoupled from any HTTP messaging client with the help of [PHP-HTTP](http://php-http.org/).
This requires another package providing [php-http/client-implementation](https://packagist.org/providers/php-http/client-implementation).
To use Guzzle 6, for example, simply require `php-http/guzzle6-adapter`:

``` bash
composer require php-http/guzzle6-adapter
```

### Laravel Service Provider

If you are using Laravel < 5.5 or have disabled package auto discover, you must add the service provider to your `config/app.php` file:

``` php
'providers' => [
    ...,
    \Swis\JsonApi\Client\Providers\ServiceProvider::class,
],
```


## Getting started

You can simply require the client as a dependency and use it in your class.
This allows you to, for example, make a repository that uses the [DocumentClient](#documentclient):

``` php
use Swis\JsonApi\Client\Interfaces\DocumentClientInterface;
use Swis\JsonApi\Client\Interfaces\ItemDocumentInterface;

class BlogRepository
{
    protected $client;

    public function __construct(DocumentClientInterface $client)
    {
        $this->client = $client;
    }

    public function all(array $parameters = [])
    {
        return $this->client->get('blogs?'.http_build_query($parameters));
    }

    public function create(ItemDocumentInterface $document, array $parameters = [])
    {
        return $this->client->post('blogs?'.http_build_query($parameters), $document);
    }

    public function find(string $id, array $parameters = [])
    {
        return $this->client->get('blogs/'.urlencode($id).'?'.http_build_query($parameters));
    }

    public function update(ItemDocumentInterface $document, array $parameters = [])
    {
        return $this->client->patch('blogs/'.urlencode($document->getData()->getId()).'?'.http_build_query($parameters), $document);
    }

    public function delete(string $id, array $parameters = [])
    {
        return $this->client->delete('blogs/'.urlencode($id).'?'.http_build_query($parameters));
    }
}
```

## Configuration

The following is the default configuration provided by this package:

``` php
return [
    /*
    |--------------------------------------------------------------------------
    | Base URI
    |--------------------------------------------------------------------------
    |
    | Specify a base uri which will be prepended to every URI.
    |
    | Default: empty string
    |
    */
    'base_uri' => '',
];
```
        
### Publish Configuration

If you would like to make changes to the default configuration, publish and edit the configuration file:

``` bash
php artisan vendor:publish --provider="Swis\JsonApi\Client\Providers\ServiceProvider" --tag="config"
```


## Clients

This package offers two clients; `DocumentClient` and `Client`.
   
### DocumentClient

This is the client that you would generally use.
Per the [JSON API spec](http://jsonapi.org/format/#document-structure), all requests and responses are documents.
Therefore, this client always expects a `\Swis\JsonApi\Client\Interfaces\DocumentInterface` as input when posting data and always returns this same interface.
This can be a plain `Document` when there is no data, an `ItemDocument` for an item, a `CollectionDocument` for a collection or an `InvalidResponseDocument` when the server responds with a non 2xx response.

The `DocumentClient` follows the following steps internally:
 1. Send the request using your HTTP client;
 2. Use [art4/json-api-client](https://github.com/art4/json-api-client) to parse and validate the response;
 3. Create the correct document instance;
 4. Hydrate every item by using the item model registered with the `TypeMapper` or a `\Swis\JsonApi\Client\Item` as fallback;
 5. Hydrate all relationships;
 6. Add meta data to the document such as [errors](http://jsonapi.org/format/#errors), [links](http://jsonapi.org/format/#document-links) and [meta](http://jsonapi.org/format/#document-meta).

### Client

This client is a more low level client and can be used, for example, for posting binary data such as images.
It can take everything your request factory takes as input data and returns the 'raw' `\Psr\Http\Message\ResponseInterface` wrapped in a `\Swis\JsonApi\Client\Response`.
It does not parse or validate the response or hydrate items!


## Items

By default, all items are an instance of `\Swis\JsonApi\Client\Item`.
The `Item` extends [jenssegers/model](https://github.com/jenssegers/model), which provides a Laravel Eloquent-like base class.
Please see it's documentation about the features it provides.
You can define your own models by extending `\Swis\JsonApi\Client\Item` or by implementing the `\Swis\JsonApi\Client\Interfaces\ItemInterface` yourself.
This can be useful if you want to define, for example, hidden attributes, casts or get/set mutators.
If you use custom models, you must register them with the [TypeMapper](#typemapper).


### Relations

On top of [jenssegers/model](https://github.com/jenssegers/model), this package has implemented [Laravel Eloquent-like relations](https://laravel.com/docs/eloquent-relationships).
These relations provide a fluent interface to retrieve the related items.
There are currently four relations available:

 * `HasOneRelation`
 * `HasManyRelation`
 * `MorphToRelation`
 * `MorphToManyRelation`

Please see the following example about defining the relationships:

``` php
use Swis\JsonApi\Client\Item;

class AuthorItem extends Item
{
    protected $type = 'author';

    public function blogs()
    {
        return $this->hasMany(BlogItem::class);
    }
}

class BlogItem extends Item
{
    protected $type = 'blog';

    public function author()
    {
        return $this->hasOne(AuthorItem::class);
    }
}
```


## Collections

This package uses [Laravel Collections](https://laravel.com/docs/collections) as a wrapper for item arrays.


## TypeMapper

All custom models must be registered with the `TypeMapper`.
This `TypeMapper` maps, as the name suggests, JSON API types to custom [item models](#item-models).

### Mapping types

You can manually register items with the `\Swis\JsonApi\Client\TypeMapper` or use the supplied `\Swis\JsonApi\Client\Providers\TypeMapperServiceProvider`:

``` php
use Swis\JsonApi\Client\Providers\TypeMapperServiceProvider as BaseTypeMapperServiceProvider;

class TypeMapperServiceProvider extends BaseTypeMapperServiceProvider
{
    /**
     * A list of class names implementing \Swis\JsonApi\Client\Interfaces\ItemInterface.
     *
     * @var string[]
     */
    protected $items = [
        \App\Items\Author::class,
        \App\Items\Blog::class,
        \App\Items\Comment::class,
    ];
}
```


## Service Provider

The `\Swis\JsonApi\Client\Providers\ServiceProvider` registers the `TypeMapper`, `JsonApi\Parser` and both clients; `Client` and `DocumentClient`.
Each section can be overwritten to allow extended customization.

### Bind TypeMapper

The service provider registers the `TypeMapper` as a singleton so your entire application has the same mappings available.

### Bind Clients

The service provider registers the `Client` and `DocumentClient` to your application.
By default it uses [php-http/discovery](https://github.com/php-http/discovery) to find a HTTP client and message factory.
You can specify your own message factory by overwriting the `getMessageFactory()` method and your own HTTP client by overwriting the `getHttpClient()` method.
The first should return a message factory and the latter should return a HTTP client, both implementing HTTPlug.
These methods are the perfect place to add extra options to your HTTP client or register a mock HTTP client for your tests:

``` php
protected function getHttpClient(): HttpClient
{
    if (app()->environment('testing')) {
        return new \Swis\Http\Fixture\Client(
            new \Swis\Http\Fixture\ResponseBuilder('/path/to/fixtures');
        );
    }

    return \Http\Adapter\Guzzle6\Client::createWithConfig(
        [
            'timeout' => 2,
        ]
    );
}
```

N.B. This example uses our [swisnl/php-http-fixture-client](https://github.com/swisnl/php-http-fixture-client) when in testing environment.
This package allows you to easily mock requests with static fixtures.
Definitely worth a try!

If you register your own service provider and use package auto discover, don't forget to exclude this package in your `package.json`:

``` json
"extra": {
  "laravel": {
    "dont-discover": [
      "swisnl/json-api-client"
    ]
  }
},
```


## Change log

Please see [CHANGELOG](CHANGELOG.md) for more information on what has changed recently.


## Testing

``` bash
composer test
```


## Contributing

Please see [CONTRIBUTING](CONTRIBUTING.md) and [CODE_OF_CONDUCT](CODE_OF_CONDUCT.md) for details.


## Security

If you discover any security related issues, please email security@swis.nl instead of using the issue tracker.


## License

The MIT License (MIT). Please see [License File](LICENSE) for more information.

## SWIS

[SWIS](https://www.swis.nl) is a web agency from Leiden, the Netherlands. We love working with open source software.
