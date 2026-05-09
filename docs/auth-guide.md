# Authentication Guide — Sanctum API Token

**Stack:** Laravel Sanctum + Vue 3 + Axios
**Method:** Token-based authentication (Personal Access Tokens)
**Features:** Register, Login, Logout, Protected Routes

---

## What Already Existed (Before This Setup)

| Feature | Status | Detail |
|---|---|---|
| Web Login/Register (Blade) | ✅ Existed | `routes/auth.php` — redirect-based, not JSON |
| Auth Controllers (web) | ✅ Existed | 9 controllers for web Breeze auth |
| Sanctum installed | ✅ Existed | `config/sanctum.php` exists |
| **Sanctum API JSON auth** | ❌ Missing | Built in this guide |
| **Vue Login/Register UI** | ❌ Missing | Built in this guide |
| **Protected `/api/users`** | ❌ Missing | Added in this guide |

---

## Architecture Overview

```
Browser opens http://user-management.local
    ↓
App.vue checks localStorage for token
    ↓
No token?              Token found?
    ↓                       ↓
Login.vue           Set axios header
or Register.vue     show UserList.vue
    ↓
User submits form
    ↓
POST /api/login  or  POST /api/register
    ↓
Laravel validates → creates Sanctum token
    ↓
Returns: { token: "...", user: {...} }
    ↓
Store token in localStorage
Set axios Authorization header
Show UserList.vue
    ↓
Every API request → Authorization: Bearer {token}
    ↓
Laravel auth:sanctum middleware verifies token
    ↓
Request allowed ✅  or  401 Unauthorized ❌
```

---

## Token Flow Explained

```
Login success
    ↓
token = "1|abc123xyz..."        ← Sanctum Personal Access Token
    ↓
localStorage.setItem('api_token', token)
    ↓
axios.defaults.headers.common['Authorization'] = `Bearer ${token}`
    ↓
Every axios request now sends:
    Authorization: Bearer 1|abc123xyz...
    ↓
Laravel reads token → finds user → allows request
```

---

## API Endpoints

| Method | Endpoint | Auth Required | Description |
|---|---|---|---|
| POST | `/api/register` | No | Create account + get token |
| POST | `/api/login` | No | Login + get token |
| POST | `/api/logout` | Yes | Revoke current token |
| GET | `/api/me` | Yes | Get logged-in user info |
| GET | `/api/users` | Yes | List users |
| POST | `/api/users` | Yes | Create user |
| PUT | `/api/users/{id}` | Yes | Update user |
| DELETE | `/api/users/{id}` | Yes | Delete user |

---

## PART 1 — Laravel

### STEP 1 — Create `ApiAuthController`

**File:** `app/Http/Controllers/Api/ApiAuthController.php`

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules;

class ApiAuthController extends Controller
{
    public function register(Request $request)
    {
        $request->validate([
            'name'     => ['required', 'string', 'max:255'],
            'email'    => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'confirmed', Rules\Password::defaults()],
        ]);

        $user = User::create([
            'name'     => $request->name,
            'email'    => $request->email,
            'password' => Hash::make($request->password),
        ]);

        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'status'  => true,
            'message' => 'Registration successful.',
            'token'   => $token,
            'user'    => ['id' => $user->id, 'name' => $user->name, 'email' => $user->email],
        ], 201);
    }

    public function login(Request $request)
    {
        $request->validate([
            'email'    => ['required', 'email'],
            'password' => ['required', 'string'],
        ]);

        if (!Auth::attempt($request->only('email', 'password'))) {
            return response()->json([
                'status'  => false,
                'message' => 'Invalid email or password.',
            ], 401);
        }

        $user  = Auth::user();
        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'status'  => true,
            'message' => 'Login successful.',
            'token'   => $token,
            'user'    => ['id' => $user->id, 'name' => $user->name, 'email' => $user->email],
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json([
            'status'  => true,
            'message' => 'Logged out successfully.',
        ]);
    }

    public function me(Request $request)
    {
        return response()->json([
            'status' => true,
            'user'   => $request->user(),
        ]);
    }
}
```

**Key methods:**

| Method | Logic |
|---|---|
| `register()` | Validate → `User::create()` → `createToken()` → return token |
| `login()` | `Auth::attempt()` → if fail return 401 → `createToken()` → return token |
| `logout()` | `currentAccessToken()->delete()` — revokes only current token |
| `me()` | Returns authenticated user from token |

---

### STEP 2 — Update `routes/api.php`

**File:** `routes/api.php`

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\ApiAuthController;
use App\Http\Controllers\Api\UserApiController;

// Public — no token needed
Route::post('/register', [ApiAuthController::class, 'register']);
Route::post('/login',    [ApiAuthController::class, 'login']);

// Protected — token required
Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [ApiAuthController::class, 'logout']);
    Route::get('/me',      [ApiAuthController::class, 'me']);

    Route::apiResource('/users', UserApiController::class);
});
```

> All `/api/users` routes are now **protected**.
> Any request without a valid token returns `401 Unauthorized`.

**Verify routes:**
```bash
php artisan route:list | grep api
```

---

### STEP 3 — Clear Cache

```bash
php artisan route:clear
php artisan config:clear
php artisan cache:clear
```

---

## PART 2 — Vue Components

### STEP 4 — How the Vue Files Are Organized

```
resources/js/
├── app.js                    ← mounts App.vue (was UserList.vue)
└── components/
    ├── App.vue               ← root: manages auth state, switches pages
    ├── Login.vue             ← login form
    ├── Register.vue          ← register form
    └── UserList.vue          ← CRUD table (protected, has logout button)
```

---

### STEP 5 — Update `app.js`

**File:** `resources/js/app.js`

```js
import './bootstrap';
import { createApp } from 'vue';
import App from './components/App.vue';

createApp(App).mount('#app');
```

> `App.vue` is now the root component — it decides which page to show.

---

### STEP 6 — `App.vue` — Auth State Manager

**File:** `resources/js/components/App.vue`

```vue
<template>
    <div>
        <Register v-if="page === 'register'" @registered="onLoggedIn" @go-login="page = 'login'" />
        <Login    v-else-if="page === 'login'" @logged-in="onLoggedIn" @go-register="page = 'register'" />
        <UserList v-else @logged-out="onLoggedOut" :auth-user="authUser" />
    </div>
</template>
```

**How it works:**

| State | Component shown |
|---|---|
| `page === 'login'` | Login.vue |
| `page === 'register'` | Register.vue |
| `page === 'app'` | UserList.vue |

**On mount — check localStorage:**
```js
onMounted(() => {
    const token = localStorage.getItem('api_token')
    if (token) {
        // Set token on all future axios requests
        axios.defaults.headers.common['Authorization'] = `Bearer ${token}`
        // Restore user info
        authUser.value = JSON.parse(localStorage.getItem('auth_user'))
        page.value = 'app'
    }
})
```

> If token exists in localStorage → user is already logged in → skip login page.

**On login/register success:**
```js
function onLoggedIn({ token, user }) {
    localStorage.setItem('api_token', token)
    localStorage.setItem('auth_user', JSON.stringify(user))
    axios.defaults.headers.common['Authorization'] = `Bearer ${token}`
    authUser.value = user
    page.value = 'app'
}
```

**On logout:**
```js
function onLoggedOut() {
    localStorage.removeItem('api_token')
    localStorage.removeItem('auth_user')
    delete axios.defaults.headers.common['Authorization']
    authUser.value = null
    page.value = 'login'
}
```

---

### STEP 7 — `Login.vue`

**File:** `resources/js/components/Login.vue`

**Form fields:** Email, Password

**On submit:**
```js
const response = await axios.post('/api/login', {
    email:    form.email,
    password: form.password,
})
emit('logged-in', { token: response.data.token, user: response.data.user })
```

**Error handling:**
| Status | Behavior |
|---|---|
| `422` | Validation errors shown under each field |
| `401` | "Invalid email or password" banner shown |
| Other | "Something went wrong" banner shown |

**Navigation:**
- "Don't have an account? **Register**" → emits `go-register` → App.vue switches to Register.vue

---

### STEP 8 — `Register.vue`

**File:** `resources/js/components/Register.vue`

**Form fields:** Full Name, Email, Password, Confirm Password

**On submit:**
```js
const response = await axios.post('/api/register', {
    name:                  form.name,
    email:                 form.email,
    password:              form.password,
    password_confirmation: form.password_confirmation,
})
emit('registered', { token: response.data.token, user: response.data.user })
```

> After successful register → user is automatically logged in (token returned immediately).

**Navigation:**
- "Already have an account? **Sign In**" → emits `go-login` → App.vue switches to Login.vue

---

### STEP 9 — UserList.vue Changes

**Added navbar** with logged-in user name + Logout button.

**Added logout function:**
```js
async function logout() {
    try {
        await axios.post('/api/logout')  // revoke token on server
    } finally {
        emit('logged-out')               // App.vue clears localStorage
    }
}
```

**Added 401 handling in `fetchUsers()`:**
```js
} catch (err) {
    if (err.response?.status === 401) {
        emit('logged-out')   // token expired → redirect to login
    } else {
        showAlert('Failed to load users.', 'error')
    }
}
```

> If token expires while user is on the page → automatically redirected to Login.

---

## PART 3 — Run & Test

### STEP 10 — Run Vite

```bash
npm run dev
```

### STEP 11 — Open Browser

```
http://user-management.local
```

**Expected flow:**
1. Login page appears
2. Register a new account or login
3. UserList page appears with navbar showing your name
4. CRUD operations work (protected by token)
5. Click Logout → back to Login page

---

## Test API with curl

**Register:**
```bash
curl -X POST http://user-management.local/api/register \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "password": "password123",
    "password_confirmation": "password123"
  }'
```

**Login:**
```bash
curl -X POST http://user-management.local/api/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@example.com","password":"password123"}'
```

**Use token to get users:**
```bash
curl http://user-management.local/api/users \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Logout:**
```bash
curl -X POST http://user-management.local/api/logout \
  -H "Authorization: Bearer YOUR_TOKEN_HERE"
```

**Access without token (should return 401):**
```bash
curl http://user-management.local/api/users
```
Expected: `{"message":"Unauthenticated."}`

---

## File Structure Summary

```
user_management/
├── app/
│   └── Http/
│       └── Controllers/
│           └── Api/
│               ├── ApiAuthController.php    ← register / login / logout / me
│               └── UserApiController.php    ← CRUD (protected)
├── resources/
│   └── js/
│       ├── app.js                           ← mounts App.vue
│       └── components/
│           ├── App.vue                      ← auth state router
│           ├── Login.vue                    ← login form
│           ├── Register.vue                 ← register form
│           └── UserList.vue                 ← CRUD + logout button
└── routes/
    └── api.php                              ← public + protected routes
```

---

## Troubleshooting

### 401 Unauthenticated on all requests
Token not being sent. Check axios header is set:
```js
axios.defaults.headers.common['Authorization'] = `Bearer ${token}`
```
Or check localStorage has the token:
```js
console.log(localStorage.getItem('api_token'))
```

### 500 on register — "createToken not found"
`HasApiTokens` trait must be on the User model:
```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
}
```

### Token not persisting after page refresh
`App.vue` `onMounted()` must read from localStorage and set axios header.
Check that `localStorage.setItem('api_token', token)` runs on login.

### `personal_access_tokens` table missing
Run migration:
```bash
php artisan migrate
```
Sanctum stores tokens in `personal_access_tokens` table.
