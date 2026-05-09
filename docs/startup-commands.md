# Startup Commands — Nginx + PHP-FPM + Laravel + Vue

ជារៀងរាល់ដងដែល restart Windows / WSL2 / Docker Desktop
ត្រូវ run commands ខាងក្រោមតាមលំដាប់នេះ។

---

## លំដាប់ Commands ពេញលេញ

### STEP 1 — Start MySQL Docker Container

```bash
docker start user_management-mysql-1
```

ពិនិត្យ container running:

```bash
docker ps | grep mysql
```

ត្រូវឃើញ status: **Up**

```
user_management-mysql-1   Up X seconds   0.0.0.0:3307->3306/tcp
```

> ⚠️ រង់ចាំ ~10 វិនាទី ឱ្យ MySQL ចាប់ផ្ដើម មុន run commands បន្ទាប់

---

### STEP 2 — Clear Laravel Cache

```bash
cd /home/dasmilan200/projects/user_management

php artisan config:clear
php artisan cache:clear
php artisan route:clear
```

> ធ្វើម្ដងម្នាក់ ឬ run ជាមួយគ្នា:
> ```bash
> php artisan config:clear && php artisan cache:clear && php artisan route:clear
> ```

---

### STEP 3 — Check PHP-FPM is Running

```bash
sudo systemctl status php8.2-fpm
```

បើ **inactive** ឬ **stopped**:

```bash
sudo systemctl start php8.2-fpm
```

---

### STEP 4 — Reload Nginx

```bash
sudo systemctl reload nginx
```

ពិនិត្យ Nginx status:

```bash
sudo systemctl status nginx
```

ត្រូវឃើញ: **active (running)**

---

### STEP 5 — Start Vite Dev Server

```bash
cd /home/dasmilan200/projects/user_management

npm run dev
```

> ⚠️ Terminal នេះ ត្រូវ keep running ពេញ session
> Vite serves JS/CSS assets + hot reload

ត្រូវឃើញ output ដូចនេះ:

```
  VITE v4.x.x  ready in xxx ms

  ➜  Local:   http://localhost:5173/
  
  Laravel plugin v0.x.x
  ➜  APP_URL: http://user-management.local
```

---

### STEP 6 — Open Browser

```
http://user-management.local
```

---

## Quick Start — Copy & Paste ទាំងអស់

បើក **2 terminals**:

**Terminal 1 — Setup:**

```bash
docker start user_management-mysql-1

sleep 10

cd /home/dasmilan200/projects/user_management

php artisan config:clear && php artisan cache:clear && php artisan route:clear

sudo systemctl start php8.2-fpm

sudo systemctl reload nginx
```

**Terminal 2 — Vite (keep running):**

```bash
cd /home/dasmilan200/projects/user_management

npm run dev
```

---

## Auto-start MySQL (មួយដង)

ដើម្បីឱ្យ MySQL container start ដោយស្វ័យប្រវត្តិ រៀងរាល់ Docker Desktop ចាប់ផ្ដើម:

```bash
docker update --restart unless-stopped user_management-mysql-1
```

---

## Troubleshooting

### 500 Internal Server Error

```bash
# ពិនិត្យ error
tail -20 /home/dasmilan200/projects/user_management/storage/logs/laravel.log

# ភាគច្រើន MySQL container stop — fix:
docker start user_management-mysql-1
sleep 10
php artisan config:clear
```

### 502 Bad Gateway

```bash
# PHP-FPM stop
sudo systemctl start php8.2-fpm
sudo systemctl reload nginx
```

### Assets មិន load (JS/CSS)

```bash
# Vite មិន run
cd /home/dasmilan200/projects/user_management
npm run dev
```

### Test DB Connection

```bash
cd /home/dasmilan200/projects/user_management
php artisan tinker --execute="echo DB::connection()->getPdo() ? 'DB OK' : 'FAIL';"
```

---

## .env Reference

```env
DB_HOST=127.0.0.1
DB_PORT=3307           ← MySQL Docker port (host port)
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password

APP_URL=http://user-management.local
```

---

## Summary Table

| Order | Command | Purpose |
|---|---|---|
| 1 | `docker start user_management-mysql-1` | Start MySQL |
| 2 | `php artisan config:clear` | Clear cached config |
| 3 | `php artisan cache:clear` | Clear app cache |
| 4 | `php artisan route:clear` | Clear route cache |
| 5 | `sudo systemctl start php8.2-fpm` | Start PHP-FPM |
| 6 | `sudo systemctl reload nginx` | Reload Nginx |
| 7 | `npm run dev` | Start Vite (keep running) |
| 8 | Open browser | `http://user-management.local` |
