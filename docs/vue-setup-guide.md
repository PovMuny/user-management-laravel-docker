# Vue.js + Laravel SPA — Complete Setup Guide

**Stack:** Nginx + PHP-FPM + Laravel + Vue 3 + MySQL (Docker)
**Domain:** `http://user-management.local`

---

## Architecture Overview

```
Browser → http://user-management.local
    ↓
Windows hosts file  (user-management.local → 127.0.0.1)
    ↓
Nginx  (/etc/nginx/sites-available/user_management)
    ↓  FastCGI
PHP-FPM
    ↓
public/index.php  →  Laravel bootstraps
    ↓
welcome.blade.php  →  loads app.js via @vite()
    ↓
Vue mounts on <div id="app">
    ↓
Vite dev server (localhost:5173) serves JS/CSS assets
    ↓
UserList.vue  →  onMounted() → axios.get('/api/users')
    ↓
Laravel routes/api.php  →  UserController
    ↓
User::latest()->get()  →  MySQL (Docker)
    ↓
JSON: { "status": true, "users": [...] }
    ↓
Vue renders table ✅
```

---

## Important Files

| File | Purpose |
|---|---|
| `routes/api.php` | Laravel API route `/api/users` |
| `resources/js/components/UserList.vue` | Vue component — table + Axios call |
| `resources/js/app.js` | Vue entry point |
| `vite.config.js` | Vite + Vue plugin config |
| `.env` | DB connection, APP_URL |
| `/etc/nginx/sites-available/user_management` | Nginx virtual host config |

---

## STEP 1 — Windows hosts file

**File:** `C:\Windows\System32\drivers\etc\hosts`

Add this line:

```
127.0.0.1   user-management.local
```

> **Why:** Windows needs to know that `user-management.local` points to your local machine (`127.0.0.1`), otherwise the browser cannot resolve the domain.

---

## STEP 2 — Nginx Virtual Host Config

**File:** `/etc/nginx/sites-available/user_management`

```nginx
server {
    listen 80;
    server_name user-management.local;

    root /home/dasmilan200/projects/user_management/public;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

**Enable the site:**

```bash
sudo ln -s /etc/nginx/sites-available/user_management /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

> **Key points:**
> - `root` must point to Laravel's `public/` folder — NOT the project root
> - `try_files` sends all requests to `index.php` so Laravel handles routing
> - `fastcgi_pass` connects Nginx to PHP-FPM via Unix socket

---

## STEP 3 — Configure `.env`

**File:** `.env`

```env
APP_NAME="User Management"
APP_ENV=local
APP_URL=http://user-management.local

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=user_management
DB_USERNAME=root
DB_PASSWORD=your_password
```

> **Important:** `APP_URL` must match the domain exactly — this is used by Laravel for asset URLs and redirects.

---

## STEP 4 — MySQL in Docker

**Start MySQL container:**

```bash
docker run -d \
  --name mysql_user_management \
  -e MYSQL_ROOT_PASSWORD=your_password \
  -e MYSQL_DATABASE=user_management \
  -p 3306:3306 \
  mysql:8.0
```

**Verify container is running:**

```bash
docker ps
```

**Test connection:**

```bash
docker exec -it mysql_user_management mysql -u root -p
```

---

## STEP 5 — Install Node Dependencies

```bash
npm install vue @vitejs/plugin-vue axios
```

**Packages:**

| Package | Purpose |
|---|---|
| `vue` | Vue.js 3 framework |
| `@vitejs/plugin-vue` | Vite plugin to compile `.vue` files |
| `axios` | HTTP client for API calls |

---

## STEP 6 — Configure `vite.config.js`

**File:** `vite.config.js`

```js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
    plugins: [
        laravel([
            'resources/css/app.css',
            'resources/js/app.js',
        ]),
        vue(),
    ],
});
```

> **Why:** Vite needs `@vitejs/plugin-vue` to compile `.vue` Single File Components (SFC). Without this, Vite cannot understand `<template>`, `<script setup>`, or `<style>` blocks.

---

## STEP 7 — Create Vue Component

**Create folder:**

```bash
mkdir -p resources/js/components
```

**File:** `resources/js/components/UserList.vue`

```vue
<template>
    <div class="p-6">
        <h1 class="text-2xl font-bold mb-4">Users</h1>

        <table class="w-full border">
            <thead>
                <tr>
                    <th class="border p-2">ID</th>
                    <th class="border p-2">Name</th>
                    <th class="border p-2">Email</th>
                </tr>
            </thead>

            <tbody>
                <tr v-for="user in users" :key="user.id">
                    <td class="border p-2">{{ user.id }}</td>
                    <td class="border p-2">{{ user.name }}</td>
                    <td class="border p-2">{{ user.email }}</td>
                </tr>
            </tbody>
        </table>
    </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const users = ref([])

onMounted(async () => {
    const response = await axios.get('/api/users')
    users.value = response.data.users
})
</script>
```

**How it works:**

| Code | Meaning |
|---|---|
| `ref([])` | Reactive variable — starts as empty array |
| `onMounted(async () => {...})` | Runs automatically when component loads |
| `axios.get('/api/users')` | HTTP GET to Laravel API |
| `users.value = response.data.users` | Stores API result into reactive variable |
| `v-for="user in users"` | Loops through array and renders each `<tr>` |

---

## STEP 8 — Update `app.js`

**File:** `resources/js/app.js`

```js
import './bootstrap';
import { createApp } from 'vue';
import UserList from './components/UserList.vue';

createApp(UserList).mount('#app');
```

**How it works:**

| Code | Meaning |
|---|---|
| `import './bootstrap'` | Loads Axios defaults (CSRF token, base URL) |
| `createApp(UserList)` | Creates Vue app with UserList as root component |
| `.mount('#app')` | Attaches Vue to `<div id="app">` in the HTML |

---

## STEP 9 — Update `welcome.blade.php`

**File:** `resources/views/welcome.blade.php`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Vue User Management</title>

    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <div id="app"></div>
</body>
</html>
```

**Key points:**

| Code | Meaning |
|---|---|
| `@vite([...])` | Blade directive — generates `<script>` and `<link>` tags pointing to Vite |
| `<div id="app">` | Empty div — Vue will render the entire UserList component here |

---

## STEP 10 — Laravel API Route

**File:** `routes/api.php`

```php
use App\Http\Controllers\UserController;

Route::get('/users', [UserController::class, 'index']);
```

**File:** `app/Http/Controllers/UserController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;

class UserController extends Controller
{
    public function index()
    {
        return response()->json([
            'status' => true,
            'users'  => User::latest()->get(),
        ]);
    }
}
```

**Expected API response:**

```json
{
    "status": true,
    "users": [
        { "id": 1, "name": "John Doe", "email": "john@example.com" },
        { "id": 2, "name": "Jane Smith", "email": "jane@example.com" }
    ]
}
```

---

## STEP 11 — Run Database Migration

```bash
php artisan migrate
```

**Seed test data (optional):**

```bash
php artisan db:seed
```

---

## STEP 12 — Run Vite Dev Server

```bash
npm run dev
```

> **Important:** Keep this terminal running at all times during development.
> Vite serves JS/CSS assets and enables Hot Module Replacement (HMR).

**Expected output:**

```
  VITE v4.x.x  ready in xxx ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose

  Laravel v10.x.x  plugin v0.x.x
  ➜  APP_URL: http://user-management.local
```

---

## STEP 13 — Open Browser

```
http://user-management.local
```

Vue.js will:
1. Load from Nginx → PHP-FPM → Laravel
2. Fetch users from `/api/users`
3. Render users dynamically in the table

---

## Full File Structure

```
user_management/
├── app/
│   └── Http/
│       └── Controllers/
│           └── UserController.php     ← returns JSON users
├── resources/
│   ├── css/
│   │   └── app.css
│   ├── js/
│   │   ├── app.js                     ← Vue entry point
│   │   ├── bootstrap.js               ← Axios + CSRF setup
│   │   └── components/
│   │       └── UserList.vue           ← table + API call
│   └── views/
│       └── welcome.blade.php          ← HTML shell with #app
├── routes/
│   └── api.php                        ← GET /api/users
├── vite.config.js                     ← Vite + Vue plugin
└── .env                               ← APP_URL, DB config

/etc/nginx/sites-available/
└── user_management                    ← Nginx virtual host
```

---

## Troubleshooting

### Nginx 502 Bad Gateway
```bash
# Check PHP-FPM is running
sudo systemctl status php8.2-fpm

# Start if not running
sudo systemctl start php8.2-fpm
```

### Vite assets not loading
```bash
# Make sure npm run dev is running
npm run dev

# Check APP_URL in .env matches domain
APP_URL=http://user-management.local
```

### API returns 404
```bash
# Clear Laravel route cache
php artisan route:clear
php artisan cache:clear

# Verify route exists
php artisan route:list | grep users
```

### MySQL connection refused
```bash
# Check Docker container is running
docker ps

# Restart MySQL container
docker start mysql_user_management
```

### Permission denied on storage/
```bash
sudo chmod -R 775 storage bootstrap/cache
sudo chown -R $USER:www-data storage bootstrap/cache
```
