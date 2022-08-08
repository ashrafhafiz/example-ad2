## Laravel/UI Active Directory Authentication

### Step 1: Create New Laravel Project

```php
composer create-project laravel/laravel example-ad2
cd example-ad2
```

### Step 2: Add Laravel UI

```php
composer require laravel/ui
```

Once the laravel/ui package has been installed, you may install the frontend scaffolding using the ui Artisan command:

```php
php artisan ui bootstrap --auth
```

### Step 3: Add LdapRecord-Laravel

```php
composer require directorytree/ldaprecord-laravel
```

#### Configuration Using an environment file (.env)

LDAP connections may be configured directly in your .env without having to publish any configuration files.
If your application has a single connection, you can paste the below env to get started right away:

```
LDAP_LOGGING=true
LDAP_CONNECTION=default
LDAP_CONNECTIONS=default

LDAP_DEFAULT_HOSTS=10.0.0.1
LDAP_DEFAULT_USERNAME="cn=admin,dc=local,dc=com"
LDAP_DEFAULT_PASSWORD=secret
LDAP_DEFAULT_PORT=389
LDAP_DEFAULT_BASE_DN="dc=local,dc=com"
LDAP_DEFAULT_TIMEOUT=5
LDAP_DEFAULT_SSL=false
LDAP_DEFAULT_TLS=false
```

#### Testing your connection

Once you have your connection(s) configured, run a quick test to make sure they've been setup properly:

```
php artisan ldap:test
```

#### Login Controller

```php
<?php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Providers\RouteServiceProvider;
use Illuminate\Foundation\Auth\AuthenticatesUsers;

use Illuminate\Http\Request;
use LdapRecord\Laravel\Auth\ListensForLdapBindFailure;

class LoginController extends Controller
{
    use AuthenticatesUsers;

    use ListensForLdapBindFailure {
        handleLdapBindError as baseHandleLdapBindError;
    }

    protected function handleLdapBindError($message, $code = null)
    {
        if ($code == '773') {
            // The users password has expired. Redirect them.
            abort(redirect('/password-reset'));
        }

        $this->baseHandleLdapBindError($message, $code);
    }


    protected $redirectTo = RouteServiceProvider::HOME;

    public function __construct()
    {
        $this->middleware('guest')->except('logout');
    }

    public function username()
    {
        return 'username';
    }

    protected function credentials(Request $request)
    {
        return [
            'samaccountname' => $request->username,
            'password' => $request->password,
        ];
    }
}
```

#### Updating Blade Views

Since an LdapRecord model instance will be returned when calling Auth::user() instead of an Eloquent model, you must change any references from:

```php
Auth::user()->name
```

To:

```php
Auth::user()->getName()
```

```html
<!-- resources/views/auth/login.blade.php -->

<!-- Before... -->
<input
    id="email"
    type="email"
    class="form-control @error('email') is-invalid @enderror"
    name="email"
    value="{{ old('email') }}"
    required
    autocomplete="email"
    autofocus
/>

<!-- After... -->
<input
    id="username"
    type="text"
    class="form-control @error('username') is-invalid @enderror"
    name="username"
    value="{{ old('username') }}"
    required
    autocomplete="username"
    autofocus
/>
```
