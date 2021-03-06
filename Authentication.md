Most APIs will usually require some form of authentication prior to returning responses. Sometimes a response may differ when a request is authenticated compared to an unauthenticated request.

This package allows you to configure multiple authentication providers. When authentication is enabled each of the providers is executed in an attempt to authenticate the request.

### Configuring Authentication Providers

By default only HTTP Basic authentication is enabled in the configuration file. Here is a list of the current supported authentication providers that are built in to the package.

- HTTP Basic (`Dingo\Api\Auth\BasicProvider`)
- JSON Web Tokens (`Dingo\Api\Auth\JWTProvider`)
- OAuth 2.0 (`Dingo\Api\Auth\LeagueOAuth2Provider`)

#### HTTP Basic

This provider is configured by default, however, if you need to configure the identifier used to authenticate a user you can do so by passing it in as the second parameter when instantiating the provider.

```php
'basic' => function ($app) {
   return new Dingo\Api\Auth\BasicProvider($app['auth'], 'email');
}
```

#### JSON Web Tokens (JWT)

This package makes use of a 3rd party package to integrate JWT authentication. Please refer to the [`tymon/jwt-auth`](https://github.com/tymondesigns/jwt-auth) GitHub page for details on installing and configuring the package.

Once you have the package you can configure the provider.

```php
'jwt' => function ($app) {
    return new Dingo\Api\Auth\JWTProvider($app['tymon.jwt.auth']);
}
```

#### OAuth 2.0

This package makes use of a 3rd party package to integrate OAuth 2.0. You can either install [`league/oauth2-server`](https://github.com/thephpleague/oauth2-server) and configure the server yourself or use the bridge package, [`lucadegasperi/oauth2-server-laravel`](https://github.com/lucadegasperi/oauth2-server-laravel).

> If using`lucadegasperi/oauth2-server-laravel` you are required to install `v3.0.*`

> For simplicity this guide will assume you are using the bridge package.

Once you have the package you can configure the provider.

```php
'oauth' => function ($app) {
    $provider = new Dingo\Api\Auth\LeagueOAuth2Provider($app['oauth2-server.authorizer']->getChecker());

    $provider->setUserResolver(function ($id) {
        // Logic to return a user by their ID.
    });

    $provider->setClientResolver(function ($id) {
        // Logic to return a client by their ID.
    });
    
    return $provider;
}
```

##### User And Client Resolvers

Depending on the authorization grants you enable you may not need both of the resolvers. If, for example, you only allow clients to authenticate via OAuth 2.0 then you are not required to set a user resolver.

The resolvers both receive the ID of the user or client and should use this ID to return an instance of the user or client. This usually involves querying the database for the user or client.

### Custom Authentication Providers

If you're developing for a legacy system or require some other form of authentication you may implement your own provider.

Your authentication provider should implement `Dingo\Api\Auth\ProviderInterface`. If authentication succeeds your provider should return an instance of the authenticated user. If authentication fails your provider should thrown a `Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException`.

```php
use Illuminate\Http\Request;
use Dingo\Api\Routing\Route;
use Dingo\Api\Auth\ProviderInterface;
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;

class CustomProvider implements ProviderInterface
{
    public function authenticate(Request $request, Route $route)
    {
        // Logic to authenticate the request.

        throw new UnauthorizedHttpException('Unable to authenticate with supplied username and password.');
    }
}
```

The abstract `Dingo\Api\Auth\AuthorizationProvider` can be extended should your provider utilize tokens sent via the `Authorization` header. The `AuthorizationProvider::validateAuthorizationHeader` method allows you to easily validate that the authorization header exists and contains a valid value.

```php

use Illuminate\Http\Request;
use Dingo\Api\Routing\Route;
use Dingo\Api\Auth\AuthorizationProvider;
use Symfony\Component\HttpKernel\Exception\BadRequestHttpException;
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;

class CustomProvider extends AuthorizationProvider
{
    public function authenticate(Request $request, Route $route)
    {
        $this->validateAuthorizationHeader($request);

        // If the authorization header passed validation we can continue to authenticate.
        // If authentication then fails we must throw the UnauthorizedHttpException.
    }

    public function getAuthorizationMethod()
    {
        return 'mac';
    }
}
```

Once you've implemented your authentication provider you can configure it.

```php
'custom' => function ($app) {
    return new CustomProvider;
}
```

### Protecting Endpoints

You can enable or disable protection at the route or group level by setting the `protected` flag to either `true` or `false` in the options.

#### Require Authentication On All Routes

```php
Route::api(['version' => 'v1', 'protected' => true], function () {
    // Routes within this group will require authentication.
});
```

#### Require Authentication On Specific Routes

```php
Route::api('v1', function () {
    Route::get('user', ['protected' => true, function () {
        // This route requires authentication.
    }]);

    Route::get('posts', function () {
        // This route does not require authentication.
    });
});
```

#### Do Not Require Authentication On Specific Routes

```php
Route::api(['version' => 'v1', 'protected' => true], function () {
    Route::get('user', function () {
        // This route requires authentication.
    });

    Route::get('posts', ['protected' => false, function () {
        // This route does not require authentication.
    });
});
```

#### Allow Only Specific Authentication Providers

If you want to set a specific authentication provider on a group of routes or specific route you can do so in the options.

```php
Route::api('v1', function () {
    Route::get('user', ['protected' => true, 'providers' => 'basic|oauth', function () {
        // This route requires authentication.
    }]);
});
```

Or you can pass an array of providers instead of a string.

#### Require Authentication On Controller Methods

If your controllers use `Dingo\Api\Routing\ControllerTrait` then you can use the `protect` and `unprotect` methods in your controllers constructor.

```php
use Dingo\Api\Routing\ControllerTrait;

class UserController extends Controller
{
    use ControllerTrait;

    public function __construct()
    {
        $this->protect('index');

        // You can also pass an array or a pipe separated string.
        $this->protect(['index', 'posts']);
        $this->protect('index|posts');

        // Do not pass in any method name to protect all methods.
        $this->protect();

        // The same rules apply to the "unprotect" method.
    }

    public function index()
    {
        //
    }

    public function posts()
    {
        //
    }
}
```

### Retrieving Authenticated User

Within a protected endpoint you can retrieve the authenticated users instance.

```php
Route::api(['version' => 'v1', 'protected' => true], function () {
    Route::get('user', function () {
        $user = API::user();

        return $user;
    });
});
```

If your controllers uses `Dingo\Api\Routing\ControllerTrait` then you can use the `$auth` property.

```php
use Dingo\Api\Routing\ControllerTrait;

class UserController extends Controller
{
    use ControllerTrait;

    public function index()
    {
        $user = $this->auth->user();

        return $user;
    }
}
```

### Optional Authentication

Sometimes you may need to adjust a response based on whether or not the request was authenticated. To do this the route should not be protected. You then simply ask for the authenticated user.

```php
Route::api(['version' => 'v1'], function () {
    Route::get('user/{id}', function ($id) {
        $user = User::findOrFail($id);

        // Attempt to authenticate the request. If the request is not authenticated
        // then we'll hide the e-mail from the response. Only authenticated
        // requests can see other users e-mails.
        if (! API::user()) {
            $hidden = $user->getHidden();

            $user->setHidden(array_merge($hidden, ['email']));
        }

        return $user;
    });
});
```

[← Transformers](https://github.com/dingo/api/wiki/Transformers) | [Rate Limiting →](https://github.com/dingo/api/wiki/Rate-Limiting)
