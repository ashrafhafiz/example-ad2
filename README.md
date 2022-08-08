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

#### Database Migration

Update the create users table migration to add the username column. The LdapRecord documentation suggests updating the email column to the username, but personally I think it’s useful to store the email in your user model. This makes it easy to use Laravel’s Notifications and Mailables.

```
php artisan make:migration add_username_column_to_users_table
```

```php
<?php
...
return new class extends Migration
{
...
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('username')->after('name')->unique();
        });
    }
...
    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('username');
        });
    }
};
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

## Stage 2: Add Database Authentication as a fallback

Synchronized Database LDAP Authentication means that an LDAP user which successfully passes LDAP authentication will be created & synchronized to your local applications database. This is helpful as you can attach typical relational database information to them, such as blog posts, attachments, etc.

#### Publishing The Required Migration

Publish the migration using the below command:

```php
php artisan vendor:publish --provider="LdapRecord\Laravel\LdapAuthServiceProvider"
```

Then run the migration using the below command:

```php
php artisan migrate
```

#### Add The Required Trait and Interface

Add the following interface and trait to your User Eloquent model:

-   Trait: LdapRecord\Laravel\Auth\AuthenticatesWithLdap
-   Interface: LdapRecord\Laravel\Auth\LdapAuthenticatable

```php
// app/User.php

// ...

use LdapRecord\Laravel\Auth\LdapAuthenticatable;
use LdapRecord\Laravel\Auth\AuthenticatesWithLdap;

class User extends Authenticatable implements LdapAuthenticatable
{
    use Notifiable, AuthenticatesWithLdap;

    // ...
}
```

#### Configuration

To configure a synchronized database LDAP authentication provider, navigate to the providers array inside of your config/auth.php file, and paste the following users provider:

```php
// config/auth.php

'providers' => [
    // ...
    'users' => [
        'driver' => 'ldap',
        'model' => LdapRecord\Models\ActiveDirectory\User::class,
        'database' => [
            'model' => App\Models\User::class,
            'sync_passwords' => true,
            'sync_attributes' => [
                'name' => 'cn',
                'email' => 'mail',
                'username' => 'samaccountname',
            ],
        ],
        'rules' => [],
    ],
],
```

#### Add Fallback to LoginController

```php
<?php

...

class LoginController extends Controller
{
...
    protected function credentials(Request $request)
    {
        return [
            'samaccountname' => $request->username,
            'password' => $request->password,
            'fallback' => [
                'username' => $request->username,
                'password' => $request->password,
            ],
        ];
    }
}
```
