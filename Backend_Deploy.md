# 🐍 FastAPI Backend Deployment Guide (Alpine Linux)

This guide explains how to deploy a FastAPI backend on an Alpine Linux VPS using Uvicorn, uv, and Supervisor. It includes setup for permanent running, auto-restart, and public URL access via Nginx.

---

## 1️⃣ Prerequisites

- Alpine Linux VPS
- Root or sudo access
- Python 3.12+ installed
- `uv` installed globally (`/usr/bin/uv`)
- Git access
- Optional: Domain pointing to VPS IP for production HTTPS

---

## 2️⃣ Directory Structure

```sh
# Bare Git repository
mkdir -p ~/apps/myapp/repo
cd ~/apps/myapp/repo
git init --bare

# Working directory
mkdir -p ~/apps/myapp/dest
```

---

## 3️⃣ Git Deployment Setup

### 3.1 Create post-receive hook

```sh
nano ~/apps/myapp/repo/hooks/post-receive
```

Paste:

```sh
#!/bin/sh
APP_DIR="$HOME/apps/myapp/dest"
GIT_DIR="$HOME/apps/myapp/repo"

echo "Deploying to $APP_DIR"

# Checkout latest code
git --work-tree=$APP_DIR --git-dir=$GIT_DIR checkout -f

cd $APP_DIR || exit

# Sync dependencies using uv
echo "Running uv sync..."
uv sync

# Restart Supervisor service
supervisorctl restart myapp

echo "Deployment finished!"
```

Make it executable:

```sh
chmod +x ~/apps/myapp/repo/hooks/post-receive
```

---

## 4️⃣ Supervisor Setup (Persistent Backend)

### 4.1 Install Supervisor

```sh
apk add supervisor
```

### 4.2 Create Supervisor config folder (if not exists)

```sh
mkdir -p /etc/supervisor.d
```

### 4.3 Create backend service config

```sh
nano /etc/supervisor.d/myapp.ini
```

Paste:

```ini
[program:myapp]
directory=/root/apps/myapp/dest
command=/usr/bin/uv run uvicorn src.main:app --host 0.0.0.0 --port 8000
autostart=true
autorestart=true
stderr_logfile=/var/log/myapp.err.log
stdout_logfile=/var/log/myapp.out.log
```

> Replace `src.main:app` with the actual module path to your FastAPI app instance.

### 4.4 Start and enable Supervisor

```sh
rc-update add supervisord
rc-service supervisord start
```

Check status:

```sh
supervisorctl status
# Expected: myapp    RUNNING
```

---

## 5️⃣ Testing Backend Locally

```sh
curl http://127.0.0.1:8000
```

✅ Should return your API response.

---

## 6️⃣ Nginx Setup (Public URL)

### 6.1 Install Nginx

```sh
apk add nginx
rc-update add nginx
rc-service nginx start
```

### 6.2 Configure Nginx

```sh
nano /etc/nginx/http.d/myapp.conf
```

Paste:

```nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

> Replace `api.yourdomain.com` with your actual domain or subdomain.

Restart Nginx:

```sh
rc-service nginx restart
```

---

## 7️⃣ HTTPS Setup (for Vercel Frontend)

Install Certbot:

```sh
apk add certbot certbot-nginx
```

Run SSL certificate:

```sh
certbot --nginx -d api.yourdomain.com
```

✅ Backend will now be accessible via:

```
https://api.yourdomain.com
```

---

## 8️⃣ CORS Setup in FastAPI

Allow your frontend domain (e.g. Vercel) to access the API:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://your-frontend.vercel.app"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

> Replace `https://your-frontend.vercel.app` with your actual frontend URL.

---

## 9️⃣ Frontend Environment Variables (Vercel)

Set the environment variable in the Vercel dashboard:

```
NEXT_PUBLIC_API_URL=https://api.yourdomain.com
```

The frontend can now call backend endpoints safely via HTTPS:

```js
fetch(`${process.env.NEXT_PUBLIC_API_URL}/endpoint`)
```

---

## 🔟 Deployment Workflow

1. Make changes locally
2. Commit and push to your remote (e.g. `vps`):

```sh
git add .
git commit -m "Your message"
git push vps main
```

The post-receive hook automatically:
- Updates code in `/dest`
- Runs `uv sync`
- Restarts the backend via Supervisor

---

## 1️⃣1️⃣ Summary

| Feature | Implementation |
|---|---|
| Always running & auto-restarts on crash/reboot | Supervisor |
| Accessible via public HTTPS URL | Nginx + Certbot |
| Compatible with Vercel frontend | CORS middleware |
| Git-based deployment | post-receive hook |
