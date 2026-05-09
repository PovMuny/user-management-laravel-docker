# Full CRUD — User Management Guide

**Stack:** Laravel API + Vue 3 + Axios
**Features:** Create, Edit, Delete, Search, Pagination

---

## What We Are Building

```
[ Search Bar                    ] [ + Add User ]

┌────┬──────────────┬───────────────────────┬─────────────┬──────────────┐
│ ID │ Name         │ Email                 │ Created     │ Actions      │
├────┼──────────────┼───────────────────────┼─────────────┼──────────────┤
│  1 │ John Doe     │ john@example.com      │ 01 Jan 2025 │ Edit  Delete │
│  2 │ Jane Smith   │ jane@example.com      │ 02 Jan 2025 │ Edit  Delete │
└────┴──────────────┴───────────────────────┴─────────────┴──────────────┘

Page 1 of 3 (25 total)        [ 1 ] [ 2 ] [ 3 ]
```

**Modals:**
- Click **+ Add User** → form modal to create
- Click **Edit** → form modal pre-filled with user data
- Click **Delete** → confirmation modal before deleting

---

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/users` | List users (search + pagination) |
| POST | `/api/users` | Create new user |
| GET | `/api/users/{id}` | Get single user |
| PUT | `/api/users/{id}` | Update user |
| DELETE | `/api/users/{id}` | Delete user |

---

## PART 1 — Laravel API

### STEP 1 — Update `routes/api.php`

**File:** `routes/api.php`

```php
<?php

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\UserApiController;

Route::apiResource('/users', UserApiController::class);

Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});
```

> `Route::apiResource` automatically registers all 5 CRUD routes:
> - `GET /api/users` → `index()`
> - `POST /api/users` → `store()`
> - `GET /api/users/{user}` → `show()`
> - `PUT /api/users/{user}` → `update()`
> - `DELETE /api/users/{user}` → `destroy()`

**Verify routes are registered:**

```bash
php artisan route:list | grep users
```

---

### STEP 2 — Update `UserApiController`

**File:** `app/Http/Controllers/Api/UserApiController.php`

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rule;

class UserApiController extends Controller
{
    public function index(Request $request)
    {
        $query = User::latest();

        if ($request->filled('search')) {
            $search = $request->search;
            $query->where(function ($q) use ($search) {
                $q->where('name', 'like', "%{$search}%")
                  ->orWhere('email', 'like', "%{$search}%");
            });
        }

        $users = $query->paginate($request->get('per_page', 10));

        return response()->json([
            'status' => true,
            'users'  => $users->items(),
            'pagination' => [
                'total'        => $users->total(),
                'per_page'     => $users->perPage(),
                'current_page' => $users->currentPage(),
                'last_page'    => $users->lastPage(),
            ],
        ]);
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'name'     => 'required|string|max:255',
            'email'    => 'required|email|unique:users,email',
            'password' => 'required|string|min:8',
        ]);

        $user = User::create([
            'name'     => $validated['name'],
            'email'    => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        return response()->json([
            'status'  => true,
            'message' => 'User created successfully.',
            'user'    => $user,
        ], 201);
    }

    public function show(User $user)
    {
        return response()->json([
            'status' => true,
            'user'   => $user,
        ]);
    }

    public function update(Request $request, User $user)
    {
        $validated = $request->validate([
            'name'     => 'required|string|max:255',
            'email'    => ['required', 'email', Rule::unique('users', 'email')->ignore($user->id)],
            'password' => 'nullable|string|min:8',
        ]);

        $user->name  = $validated['name'];
        $user->email = $validated['email'];

        if (!empty($validated['password'])) {
            $user->password = Hash::make($validated['password']);
        }

        $user->save();

        return response()->json([
            'status'  => true,
            'message' => 'User updated successfully.',
            'user'    => $user,
        ]);
    }

    public function destroy(User $user)
    {
        $user->delete();

        return response()->json([
            'status'  => true,
            'message' => 'User deleted successfully.',
        ]);
    }
}
```

**How each method works:**

| Method | Logic |
|---|---|
| `index()` | Search by name/email using `LIKE`, paginate 10 per page |
| `store()` | Validate → `Hash::make(password)` → `User::create()` |
| `show()` | Route model binding — Laravel auto-fetches user by ID |
| `update()` | `Rule::unique()->ignore($user->id)` — allows same email on update |
| `destroy()` | Delete user by ID |

---

### STEP 3 — Test API with curl

**Get all users:**
```bash
curl http://user-management.local/api/users
```

**Search:**
```bash
curl "http://user-management.local/api/users?search=john"
```

**Create user:**
```bash
curl -X POST http://user-management.local/api/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com","password":"password123"}'
```

**Update user:**
```bash
curl -X PUT http://user-management.local/api/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"John Updated","email":"john@example.com"}'
```

**Delete user:**
```bash
curl -X DELETE http://user-management.local/api/users/1
```

---

## PART 2 — Vue Frontend

### STEP 4 — Update `UserList.vue`

**File:** `resources/js/components/UserList.vue`

The component is split into 3 logical sections:

#### Section A — Template (UI)

```
1. Header bar          → title + "+ Add User" button
2. Search input        → debounced 400ms
3. Alert message       → success / error banner
4. Users table         → v-for loop with Edit/Delete buttons
5. Pagination          → page number buttons
6. Create/Edit modal   → shared form modal
7. Delete modal        → confirmation dialog
```

#### Section B — State (reactive data)

```js
const users      = ref([])          // array of users from API
const loading    = ref(false)       // shows "Loading..." in table
const submitting = ref(false)       // disables button during API call
const search     = ref('')          // search input value
const pagination = reactive({...})  // total, current_page, last_page
const alert      = reactive({...})  // message + type (success/error)
const modal      = reactive({...})  // show (bool) + mode (create/edit)
const form       = reactive({...})  // id, name, email, password
const errors     = ref({})          // Laravel validation errors
const deleteModal = reactive({...}) // show (bool) + user (object)
```

#### Section C — Functions

| Function | Purpose |
|---|---|
| `fetchUsers(page)` | GET `/api/users?page=&search=` |
| `onSearch()` | Debounced search — waits 400ms before calling API |
| `goToPage(page)` | Calls `fetchUsers(page)` |
| `openCreateModal()` | Resets form, opens modal in create mode |
| `openEditModal(user)` | Pre-fills form with user data, opens modal in edit mode |
| `submitForm()` | POST (create) or PUT (update) depending on `modal.mode` |
| `confirmDelete(user)` | Opens delete confirmation modal |
| `deleteUser()` | DELETE `/api/users/{id}` |
| `showAlert(msg, type)` | Shows banner, auto-hides after 4 seconds |
| `formatDate(dateStr)` | Formats `2025-01-01T00:00:00Z` → `01 Jan 2025` |

---

### STEP 5 — How Search + Pagination Works

#### Search Flow

```
User types in search box
    ↓
onSearch() fires
    ↓
clearTimeout (cancel previous timer)
    ↓
setTimeout 400ms
    ↓
fetchUsers(page=1) called
    ↓
GET /api/users?search=john&page=1
    ↓
Laravel: WHERE name LIKE '%john%' OR email LIKE '%john%'
    ↓
Table updates with filtered results
```

> **Why debounce?** Without debounce, every keystroke sends an API request.
> With 400ms delay, we wait until user stops typing — much more efficient.

#### Pagination Flow

```
User clicks page number [2]
    ↓
goToPage(2)
    ↓
fetchUsers(page=2)
    ↓
GET /api/users?page=2
    ↓
Laravel: paginate(10) with offset
    ↓
Table shows records 11-20
    ↓
pagination.current_page = 2 (active button highlighted)
```

---

### STEP 6 — How Create User Works

```
Click "+ Add User"
    ↓
openCreateModal() → clears form, modal.mode = 'create'
    ↓
User fills: Name, Email, Password
    ↓
Click "Create User"
    ↓
submitForm() → POST /api/users { name, email, password }
    ↓
Laravel validates → User::create() → Hash::make(password)
    ↓
Returns: { status: true, user: {...} }
    ↓
closeModal() → fetchUsers() → showAlert("User created")
```

**Validation errors displayed inline:**

```
If email is already taken → errors.email[0] shown under input field
If password < 8 chars    → errors.password[0] shown under input field
```

---

### STEP 7 — How Edit User Works

```
Click "Edit" on a row
    ↓
openEditModal(user) → pre-fills form with user data
modal.mode = 'edit', form.id = user.id
    ↓
User edits fields (password optional — leave blank to keep)
    ↓
Click "Save Changes"
    ↓
submitForm() → PUT /api/users/{id} { name, email, password? }
    ↓
Laravel: Rule::unique()->ignore(id) → allows same email
password only updated if not blank
    ↓
Returns: { status: true, user: {...} }
    ↓
closeModal() → fetchUsers() → showAlert("User updated")
```

---

### STEP 8 — How Delete User Works

```
Click "Delete" on a row
    ↓
confirmDelete(user) → deleteModal.show = true
    ↓
Modal shows: "Are you sure you want to delete John Doe?"
    ↓
Click "Yes, Delete"
    ↓
deleteUser() → DELETE /api/users/{id}
    ↓
Returns: { status: true, message: "User deleted" }
    ↓
deleteModal.show = false → fetchUsers() → showAlert("User deleted")
```

> **Smart pagination on delete:** If user deletes the last item on a page (e.g., only 1 user on page 3), Vue automatically goes back to page 2.

---

## PART 3 — Run the Application

### STEP 9 — Clear Laravel Cache

```bash
php artisan route:clear
php artisan cache:clear
php artisan config:clear
```

### STEP 10 — Run Vite Dev Server

```bash
npm run dev
```

> Keep terminal running. Vite serves assets and enables hot-reload.

### STEP 11 — Open Browser

```
http://user-management.local
```

---

## API Response Reference

### GET `/api/users`

```json
{
    "status": true,
    "users": [
        { "id": 1, "name": "John Doe", "email": "john@example.com", "created_at": "2025-01-01T00:00:00Z" }
    ],
    "pagination": {
        "total": 25,
        "per_page": 10,
        "current_page": 1,
        "last_page": 3
    }
}
```

### POST `/api/users` — Validation Error (422)

```json
{
    "message": "The email has already been taken.",
    "errors": {
        "email": ["The email has already been taken."],
        "password": ["The password field is required."]
    }
}
```

### DELETE `/api/users/{id}`

```json
{
    "status": true,
    "message": "User deleted successfully."
}
```

---

## File Structure

```
user_management/
├── app/
│   └── Http/
│       └── Controllers/
│           └── Api/
│               └── UserApiController.php   ← CRUD + Search + Pagination
├── resources/
│   └── js/
│       └── components/
│           └── UserList.vue                ← Full CRUD Vue UI
└── routes/
    └── api.php                             ← Route::apiResource('/users', ...)
```

---

## Troubleshooting

### 405 Method Not Allowed on PUT/DELETE
Laravel requires `PUT` and `DELETE` requests from HTML forms to use `_method` spoofing — but since we are using **Axios (not HTML forms)**, this is not needed. Axios sends real `PUT` and `DELETE` requests.

If 405 persists, check routes:
```bash
php artisan route:list | grep users
```

### 419 CSRF Token Mismatch
Axios must send CSRF token. This is handled by `bootstrap.js` automatically via:
```js
window.axios.defaults.headers.common['X-CSRF-TOKEN'] = document.querySelector('meta[name="csrf-token"]').content;
```
Make sure `welcome.blade.php` has:
```html
<meta name="csrf-token" content="{{ csrf_token() }}">
```

### Validation errors not showing
Check that `errors.value` is being populated from `err.response.data.errors`:
```js
if (err.response?.status === 422) {
    errors.value = err.response.data.errors
}
```

---

### Buttons / Colors not showing (Tailwind classes missing)

**Symptom:** Table loads, data shows, but Edit/Delete buttons are invisible — no color, no background.

**Root cause:**

Tailwind CSS only generates styles for classes it can **find by scanning files**.
By default, Laravel's `tailwind.config.js` only scans `.blade.php` files.
Vue component files (`.vue`) are **not scanned** — so all classes used inside them are removed from the final CSS output.

```
tailwind.config.js  →  content: ['.../*.blade.php']
                              ↓
Tailwind scans ONLY blade files
                              ↓
bg-yellow-400, bg-red-500, text-white  ← used in .vue files
NOT FOUND → REMOVED from CSS
                              ↓
Buttons render in HTML but have no style → INVISIBLE
```

**Fix — Update `tailwind.config.js`:**

**File:** `tailwind.config.js`

```js
import defaultTheme from 'tailwindcss/defaultTheme';
import forms from '@tailwindcss/forms';

/** @type {import('tailwindcss').Config} */
export default {
    content: [
        './vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php',
        './storage/framework/views/*.php',
        './resources/views/**/*.blade.php',
        './resources/js/**/*.vue',   // ← ADD THIS
        './resources/js/**/*.js',    // ← ADD THIS
    ],

    theme: {
        extend: {
            fontFamily: {
                sans: ['Figtree', ...defaultTheme.fontFamily.sans],
            },
        },
    },

    plugins: [forms],
};
```

**After saving, restart Vite:**

```bash
# Stop current Vite process (Ctrl+C), then:
npm run dev
```

Tailwind will now scan all `.vue` and `.js` files and generate the correct CSS for every class used in Vue components.

**Result:**

```
After fix:
content: ['.../*.blade.php', '.../*.vue', '.../*.js']
                              ↓
Tailwind finds bg-yellow-400, bg-red-500, text-white...
                              ↓
CSS generated correctly
                              ↓
Edit button  → yellow background ✅
Delete button → red background  ✅
```

> **Rule to remember:** Every time you add a new CSS framework or a new file type (`.vue`, `.jsx`, `.ts`), you must add it to the `content` array in `tailwind.config.js`. Otherwise Tailwind will not generate styles for classes used in those files.
