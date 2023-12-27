# 
[Source](https://jurin.medium.com/securing-laravel-10-api-using-jwt-a5b6dca58fd7)

## Step 1 - Download the necessary library

```
composer require php-open-source-saver/jwt-auth
```

## Step 2 - Publish its stuff

```php
php artisan vendor:publish --provider="PHPOpenSourceSaver\JWTAuth\Providers\LaravelServiceProvider"
```

## Step 3 - Generate JWT secret key

```
php artisan jwt:secret
```

## Step 4 - Setting up the Auth.php

Open `auth.php`config, add the following code in guards section.

```php
'guards' => [
    ...,
    'api' => [
        'driver' => 'jwt',
        'provider' => 'users',
    ],
]
```

## Step 5 - Update User.php

```php
use PHPOpenSourceSaver\JWTAuth\Contracts\JWTSubject;

class User extends Authenticatable implements JWTSubject
{

   public function getJWTIdentifier()
    {
      return $this->getKey();
    }

    public function getJWTCustomClaims()
    {
      return [
        'email'=>$this->email,
        'name'=>$this->name
      ];
    }
}
```

## Step 6 - Create the Auth Controller

```
php artisan make:controller AuthController
```

## Step 7 - Setup the Auth Controller

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;

class AuthController extends Controller
{

    public function __construct()
    {
        $this->middleware('auth:api', ['except' => ['login','register','refresh','logout']]);
    }

    public function register(Request $request){
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:6',
        ]);

        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        $token = Auth::guard('api')->login($user);
        return response()->json([
            'status' => 'success',
            'message' => 'User created successfully',
            'user' => $user,
            'authorisation' => [
                'token' => $token,
                'type' => 'bearer',
            ]
        ]);
    }

    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|string|email',
            'password' => 'required|string',
        ]);
        $credentials = $request->only('email', 'password');

        $token = Auth::guard('api')->attempt($credentials);
        if (!$token) {
            return response()->json([
                'status' => 'error',
                'message' => 'Unauthorized',
            ], 401);
        }

        $user = Auth::guard('api')->user();
        return response()->json([
                'status' => 'success',
                'user' => $user,
                'authorisation' => [
                    'token' => $token,
                    'type' => 'bearer',
                ]
            ]);
    }

    public function logout()
    {
        Auth::guard('api')->logout();
        return response()->json([
            'status' => 'success',
            'message' => 'Successfully logged out',
        ]);
    }

    public function refresh()
    {
        return response()->json([
            'status' => 'success',
            'user' => Auth::guard('api')->user(),
            'authorisation' => [
                'token' => Auth::guard('api')->refresh(),
                'type' => 'bearer',
            ]
        ]);
    }
}
```

## Step 8 - Setup the routes for api auth

```php
use App\Http\Controllers\AuthController;

Route::post('register',[AuthController::class,'register']);
Route::post('login', [AuthController::class,'login']);
Route::post('refresh', [AuthController::class,'refresh']);
Route::post('logout', [AuthController::class,'logout']);
```

## Step 9 - Start the server

```
php artisan serve
```

## Step 10 - Testing the api

### register


```
POST http://localhost:8000/api/register
```

**Request body:**

```json
{
    "name": "Admin",
    "email": "admin@localhost",
    "password": "ArEm-o9Rpi!ENdE!"
}
```

**Response body:**

```json
{
    "status": "success",
    "message": "User created successfully",
    "user": {
        "name": "Admin",
        "email": "admin@localhost",
        "updated_at": "2023-12-27T23:43:59.000000Z",
        "created_at": "2023-12-27T23:43:59.000000Z",
        "id": 1
    },
    "authorisation": {
        "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwMDAvYXBpL3JlZ2lzdGVyIiwiaWF0IjoxNzAzNzIwNjM5LCJleHAiOjE3MDM3MjQyMzksIm5iZiI6MTcwMzcyMDYzOSwianRpIjoiVldJM050d0tWZXBnV05MZiIsInN1YiI6IjEiLCJwcnYiOiIyM2JkNWM4OTQ5ZjYwMGFkYjM5ZTcwMWM0MDA4NzJkYjdhNTk3NmY3IiwiZW1haWwiOiJhZG1pbkBsb2NhbGhvc3QiLCJuYW1lIjoiQWRtaW4ifQ.gDrYdq7W6iFouqwhW0aZPNbxZ4PyvRztHzivq1Np3pU",
        "type": "bearer"
    }
}
```

### login

```
POST http://localhost:8000/api/login
```

**Request body:**

```json
{
    "email": "admin@localhost",
    "password": "ArEm-o9Rpi!ENdE!"
}
```

**Response body:**

```json
{
    "status": "success",
    "user": {
        "id": 1,
        "name": "Admin",
        "email": "admin@localhost",
        "email_verified_at": "2023-12-27T23:36:31.000000Z",
        "created_at": "2023-12-27T23:36:31.000000Z",
        "updated_at": "2023-12-27T23:36:31.000000Z"
    },
    "authorisation": {
        "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwMDAvYXBpL2xvZ2luIiwiaWF0IjoxNzAzNzIwMjE4LCJleHAiOjE3MDM3MjM4MTgsIm5iZiI6MTcwMzcyMDIxOCwianRpIjoiUmRkajdNaTFNSTJRcXJhaSIsInN1YiI6IjEiLCJwcnYiOiIyM2JkNWM4OTQ5ZjYwMGFkYjM5ZTcwMWM0MDA4NzJkYjdhNTk3NmY3IiwiZW1haWwiOiJhZG1pbkBsb2NhbGhvc3QiLCJuYW1lIjoiQWRtaW4ifQ.kMB5KbUfNqcNDcAkNBSLDEeD6hO0ZjhSSN9NsSK0JiI",
        "type": "bearer"
    }
}
```


### logout

```
POST http://localhost:8000/api/logout
```

**Request header:**

```
Authentication: bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwMDAvYXBpL2xvZ2luIiwiaWF0IjoxNzAzNzIwMjE4LCJleHAiOjE3MDM3MjM4MTgsIm5iZiI6MTcwMzcyMDIxOCwianRpIjoiUmRkajdNaTFNSTJRcXJhaSIsInN1YiI6IjEiLCJwcnYiOiIyM2JkNWM4OTQ5ZjYwMGFkYjM5ZTcwMWM0MDA4NzJkYjdhNTk3NmY3IiwiZW1haWwiOiJhZG1pbkBsb2NhbGhvc3QiLCJuYW1lIjoiQWRtaW4ifQ.kMB5KbUfNqcNDcAkNBSLDEeD6hO0ZjhSSN9NsSK0JiI
```

***No Request body***


**Response body:**

```json
{
    "status": "success",
    "user": {
        "id": 1,
        "name": "Admin",
        "email": "admin@localhost",
        "email_verified_at": null,
        "created_at": "2023-12-27T23:43:59.000000Z",
        "updated_at": "2023-12-27T23:43:59.000000Z"
    },
    "authorisation": {
        "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwMDAvYXBpL2xvZ2luIiwiaWF0IjoxNzAzNzIwOTIxLCJleHAiOjE3MDM3MjQ1MjEsIm5iZiI6MTcwMzcyMDkyMSwianRpIjoiV1dLVXQ4SUMzU2N5dTFObSIsInN1YiI6IjEiLCJwcnYiOiIyM2JkNWM4OTQ5ZjYwMGFkYjM5ZTcwMWM0MDA4NzJkYjdhNTk3NmY3IiwiZW1haWwiOiJhZG1pbkBsb2NhbGhvc3QiLCJuYW1lIjoiQWRtaW4ifQ.hxnAhArzC-FW76rLJXhTI8ChfBYWd19S5fOz2B93NLo",
        "type": "bearer"
    }
}
```

## Step 11 - Add dependency to the models

```php
    public function __construct()
    {
        $this->middleware('auth:api');
    }
```
