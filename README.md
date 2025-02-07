# Instagram for Business Provider for OAuth 2.0 Client

[![Latest Stable Version](https://poser.pugx.org/janyksteenbeek/league-oauth2-instagram-business/v/stable.png)](https://packagist.org/packages/janyksteenbeek/league-oauth2-instagram-business)

This package provides Instagram Business OAuth 2.0 support for the PHP League's [OAuth 2.0 Client](https://github.com/thephpleague/oauth2-client). It is using the Facebook login flow but for Instagram Business & Creator accounts.

This package is compliant with [PSR-1][], [PSR-2][], [PSR-4][], and [PSR-7][]. If you notice compliance oversights,
please send a patch via pull request.

[PSR-1]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-1-basic-coding-standard.md
[PSR-2]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-2-coding-style-guide.md
[PSR-4]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-4-autoloader.md
[PSR-7]: https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-7-http-message.md


## Requirements

The following versions of PHP are supported.

* PHP 8.0
* PHP 8.1
* PHP 8.2

## Installation

Add the following to your `composer.json` file.

```json
{
    "require": {
        "janyksteenbeek/league-oauth2-instagram-business": "^1.0"
    }
}
```

## Usage

### Authorization Code Flow

```php
session_start();

$provider = new \League\OAuth2\Client\Provider\Instagram([
    'clientId'          => '{facebook-app-id}',
    'clientSecret'      => '{facebook-app-secret}',
    'redirectUri'       => 'https://example.com/callback-url',
    'graphApiVersion'   => 'v2.10',
]);

if (!isset($_GET['code'])) {

    // If we don't have an authorization code then get one
    $authUrl = $provider->getAuthorizationUrl([
        'scope' => ['email', '...', '...'],
    ]);
    $_SESSION['oauth2state'] = $provider->getState();
    
    echo '<a href="'.$authUrl.'">Log in with Instagram</a>';
    exit;

// Check given state against previously stored one to mitigate CSRF attack
} elseif (empty($_GET['state']) || ($_GET['state'] !== $_SESSION['oauth2state'])) {

    unset($_SESSION['oauth2state']);
    echo 'Invalid state.';
    exit;

}

// Try to get an access token (using the authorization code grant)
$token = $provider->getAccessToken('authorization_code', [
    'code' => $_GET['code']
]);

// Optional: Now you have a token you can look up a users profile data
try {

    // We got an access token, let's now get the user's details
    $user = $provider->getResourceOwner($token);

    // Use these details to create a new profile
    printf('Hello @%s!', $user->getUsername());
    
    echo '<pre>';
    var_dump($user);
    # object(League\OAuth2\Client\Provider\InstagramUser)#10 (1) { ...
    echo '</pre>';

} catch (\Exception $e) {

    // Failed to get user details
    exit('Oh dear...');
}

echo '<pre>';
// Use this to interact with an API on the users behalf
var_dump($token->getToken());
# string(217) "CAADAppfn3msBAI7tZBLWg...

// The time (in epoch time) when an access token will expire
var_dump($token->getExpires());
# int(1436825866)
echo '</pre>';
```

### The FacebookUser Entity

When using the `getResourceOwner()` method to obtain the user node, it will be returned as a `InstagramUser` entity.

```php
$user = $provider->getResourceOwner($token);

$id = $user->getId();
var_dump($id);
# string(1) "4"

$name = $user->getName();
var_dump($name);
# string(15) "Mark Zuckerberg"

$firstName = $user->getFirstName();
var_dump($firstName);
# string(4) "Mark"

$lastName = $user->getLastName();
var_dump($lastName);
# string(10) "Zuckerberg"

# Requires the "email" permission
$email = $user->getEmail();
var_dump($email);
# string(15) "thezuck@foo.com"

# Requires the "user_hometown" permission
$hometown = $user->getHometown();
var_dump($hometown);
# array(10) { ["id"]=> string(10) "12345567890" ...

# Requires the "user_about_me" permission
$bio = $user->getBio();
var_dump($bio);
# string(426) "All about me...

$pictureUrl = $user->getPictureUrl();
var_dump($pictureUrl);
# string(224) "https://fbcdn-profile-a.akamaihd.net/hprofile- ...

$isDefaultPicture = $user->isDefaultPicture();
var_dump($isDefaultPicture);
# boolean false

$coverPhotoUrl = $user->getCoverPhotoUrl();
var_dump($coverPhotoUrl);
# string(111) "https://fbcdn-profile-a.akamaihd.net/hphotos- ...

$gender = $user->getGender();
var_dump($gender);
# string(4) "male"

$locale = $user->getLocale();
var_dump($locale);
# string(5) "en_US"

$timezone = $user->getTimezone();
var_dump($timezone);
# int -5

$link = $user->getLink();
var_dump($link);
# string(62) "https://www.facebook.com/app_scoped_user_id/1234567890/"

$maxAge = $user->getMaxAge();
var_dump($maxAge);
# int 17 | null

$minAge = $user->getMinAge();
var_dump($minAge);
# int 21
```

You can also get all the data from the User node as a plain-old PHP array with `toArray()`.

```php
$userData = $user->toArray();
```

### Graph API Version

The `graphApiVersion` option is required. If it is not set, an `\InvalidArgumentException` will be thrown.

```php
$provider = new League\OAuth2\Client\Provider\Facebook([
    /* . . . */
    'graphApiVersion'   => 'v2.10',
]);
```

Each version of the Graph API has breaking changes from one version to the next. This package no longer supports a fallback to a default Graph version since your app might break when the fallback Graph version is updated.

See the [Graph API version schedule](https://developers.facebook.com/docs/apps/changelog) for more info.

### Beta Tier

Facebook has a [beta tier](https://developers.facebook.com/docs/apps/beta-tier) that contains the latest deployments before they are rolled out to production. To enable the beta tier, set the `enableBetaTier` option to `true`.

```php
$provider = new League\OAuth2\Client\Provider\Facebook([
    /* . . . */
    'enableBetaTier'   => true,
]);
```

### Refreshing a Token

Facebook does not support refreshing tokens. In order to get a new "refreshed" token, you must send the user through the login with Instagram process again.

From the [Facebook documentation](https://developers.facebook.com/docs/facebook-login/access-tokens#extending):

> Once [the access tokens] expire, your app must send the user through the login flow again to generate a new short-lived token.

The following code will throw a `League\OAuth2\Client\Provider\Exception\InstagramProviderException`.

```php
$grant = new \League\OAuth2\Client\Grant\RefreshToken();
$token = $provider->getAccessToken($grant, ['refresh_token' => $refreshToken]);
```

### Long-lived Access Tokens

Facebook will allow you to extend the lifetime of an access token by [exchanging a short-lives access token with a long-lived access token](https://developers.facebook.com/docs/facebook-login/access-tokens#extending).

Once you obtain a short-lived (default) access token, you can exchange it for a long-lived one.

```php
try {
    $token = $provider->getLongLivedAccessToken('short-lived-access-token');
} catch (Exception $e) {
    echo 'Failed to exchange the token: '.$e->getMessage();
    exit();
}

var_dump($token->getToken());
# string(217) "CAADAppfn3msBAI7tZBLWg...
```

### Getting Additional Data

Once you've obtained a user access token you can make additional requests to the Graph API using your [favorite HTTP client](https://github.com/guzzle/guzzle) to send the requests. For this example, we'll just use PHP's built-in `file_get_contents()` as our HTTP client to grab 5 events from the the authenticated user.

```php
// Get 5 events from authenticated user
// Requires the `user_events` permission
$baseUrl = 'https://graph.facebook.com/v2.10';
$params = http_build_query([
    'fields' => 'id,name,start_time',
    'limit' => '5',
    'access_token' => $token->getToken(),
    'appsecret_proof' => hash_hmac('sha256', $token->getToken(), '{facebook-app-secret}'),
]);
$response = file_get_contents($baseUrl.'/me/events?'.$params);

// Raw JSON response from the Graph API
var_dump($response);
# string(1190) "{"data":[{"id":"123","name":"Derby City Swing 2016","start_time":"2016-01-28T17:00:00-0500"} ...

// Response as a plain-old PHP array
$data = json_decode($response, true);
var_dump($data);
# array(2) { ["data"]=> array(5) { ...
```

See more about:

- [The `/{user-id}/events` edge](https://developers.facebook.com/docs/graph-api/reference/user/events).
- [The `appsecret_proof`](https://developers.facebook.com/docs/graph-api/securing-requests).
- [The `file_get_contents()` function](http://php.net/file_get_contents).

If you need to make even more complex queries to the Graph API to get lots of data back with just one request, check out the [Facebook Query Builder](https://github.com/SammyK/FacebookQueryBuilder).

## Testing

``` bash
$ ./vendor/bin/phpunit
```

## Credits

- We thank [Sammy Kaye Powers](https://github.com/SammyK) for his work on the original Facebook provider
- [All Contributors](https://github.com/janyksteenbeek/league-oauth2-instagram-business/contributors)


## License

The MIT License (MIT). Please see [License File](https://github.com/janyksteenbeek/league-oauth2-instagram-business/blob/master/LICENSE) for more information.
