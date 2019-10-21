# Ozwillo Provider for OAuth 2.0 Client

This package provides Ozwillo OAuth 2.0 support for the PHP League's [OAuth 2.0 Client](https://github.com/thephpleague/oauth2-client).

According to the doc: https://doc.ozwillo.com

This package is compliant with [PSR-1][], [PSR-2][] and [PSR-4][]. If you notice compliance oversights, please send
a patch via pull request.

[PSR-1]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md
[PSR-2]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md
[PSR-4]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md

## Requirements

The following versions of PHP are supported.

* PHP 7.0
* PHP 7.1
* PHP 7.2
* PHP 7.3

This package uses [OpenID Connect][openid-connect] to authenticate users with
Ozwillo accounts.

To use this package, it will be necessary to have an Ozwillo client ID and client
secret. These are referred to as `{ozwillo-client-id}` and `{ozwillo-client-secret}`
in the documentation.

## Installation

To install, use composer:

```sh
composer require angelsbaytech/oauth2-ozwillo
```

## Usage

### Authorization Code Flow

```php
use League\OAuth2\Client\Provider\Ozwillo;

$provider = new Ozwillo([
    'clientId'     => '{ozwillo-client-id}',
    'clientSecret' => '{ozwillo-client-secret}',
    'redirectUri'  => 'https://example.com/callback-url',
    'hostedDomain' => 'example.com', // optional; used to restrict access to users on your G Suite/Ozwillo Apps for Business accounts
]);

if (!empty($_GET['error'])) {

    // Got an error, probably user denied access
    exit('Got error: ' . htmlspecialchars($_GET['error'], ENT_QUOTES, 'UTF-8'));

} elseif (empty($_GET['code'])) {

    // If we don't have an authorization code then get one
    $authUrl = $provider->getAuthorizationUrl();
    $_SESSION['oauth2state'] = $provider->getState();
    header('Location: ' . $authUrl);
    exit;

} elseif (empty($_GET['state']) || ($_GET['state'] !== $_SESSION['oauth2state'])) {

    // State is invalid, possible CSRF attack in progress
    unset($_SESSION['oauth2state']);
    exit('Invalid state');

} else {

    // Try to get an access token (using the authorization code grant)
    $token = $provider->getAccessToken('authorization_code', [
        'code' => $_GET['code']
    ]);

    // Optional: Now you have a token you can look up a users profile data
    try {

        // We got an access token, let's now get the owner details
        $ownerDetails = $provider->getResourceOwner($token);

        // Use these details to create a new profile
        printf('Hello %s!', $ownerDetails->getFirstName());

    } catch (Exception $e) {

        // Failed to get user details
        exit('Something went wrong: ' . $e->getMessage());

    }

    // Use this to interact with an API on the users behalf
    echo $token->getToken();

    // Use this to get a new access token if the old one expires
    echo $token->getRefreshToken();

    // Unix timestamp at which the access token expires
    echo $token->getExpires();
}
```

#### Available Options

The `Ozwillo` provider has the following [options][auth-params]:

- `accessType` to use online or offline access
- `hostedDomain` to authenticate G Suite users
- `prompt` to modify the prompt that the user will see
- `scopes` to request access to additional user information


#### Accessing Token JWT

Ozwillo provides a [JSON Web Token][jwt] (JWT) with all access tokens. This token
[contains basic information][openid-jwt] about the authenticated user. The JWT
can be accessed from the `id_token` value of the access token:

```php
/** @var League\OAuth2\Client\Token\AccessToken $token */
$values = $token->getValues();

/** @var string */
$jwt = $values['id_token'];
```

Parsing the JWT will require a [JWT parser][jwt-parsers]. Refer to parser
documentation for instructions.

[jwt]: https://jwt.io/
[jwt-parsers]: https://packagist.org/search/?q=jwt

### Refreshing a Token

Refresh tokens are only provided to applications which request offline access. You can specify offline access by setting the `accessType` option in your provider:

```php
use League\OAuth2\Client\Provider\Ozwillo;

$provider = new Ozwillo([
    'clientId'     => '{ozwillo-client-id}',
    'clientSecret' => '{ozwillo-client-secret}',
    'redirectUri'  => 'https://example.com/callback-url',
    'accessType'   => 'offline',
]);
```

It is important to note that the refresh token is only returned on the first request after this it will be `null`. You should securely store the refresh token when it is returned:

```php
$token = $provider->getAccessToken('authorization_code', [
    'code' => $code
]);

// persist the token in a database
$refreshToken = $token->getRefreshToken();
```

If you ever need to get a new refresh token you can request one by forcing the consent prompt:

```php
$authUrl = $provider->getAuthorizationUrl(['prompt' => 'consent']);
```

Now you have everything you need to refresh an access token using a refresh token:

```php
use League\OAuth2\Client\Provider\Ozwillo;
use League\OAuth2\Client\Grant\RefreshToken;

$provider = new Ozwillo([
    'clientId'     => '{ozwillo-client-id}',
    'clientSecret' => '{ozwillo-client-secret}',
    'redirectUri'  => 'https://example.com/callback-url',
]);

$grant = new RefreshToken();
$token = $provider->getAccessToken($grant, ['refresh_token' => $refreshToken]);
```

## Scopes

Additional [scopes][scopes] can be set by using the `scope` parameter when
generating the authorization URL:

```php
$authorizationUrl = $provider->getAuthorizationUrl([
    'scope' => [
        'scope-url-here'
    ],
]);
```

## Testing

Tests can be run with:

```sh
composer test
```

Style checks can be run with:

```sh
composer check
```

## License

The MIT License (MIT). Please see [License File](https://github.com/angelsbaytech/oauth2-ozwillo/blob/master/LICENSE) for more information.
