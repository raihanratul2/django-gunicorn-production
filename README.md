# Django Production Deployment (Django + Gunicorn + Nginx)

This guide explains how to deploy a Django project in production using **Gunicorn** and **Nginx** on an Ubuntu server with best practices.

---

# 1. Server Preparation

Update the server and install required packages.

```bash
sudo apt update
sudo apt upgrade -y
```

```bash
sudo apt install python3 python3-venv python3-pip nginx git -y
```

Check installations:

```bash
python3 --version
nginx -v
```

---

# 2. Project Setup

Clone your repository.

```bash
cd /var/www/
git clone https://github.com/raihanratul2/REPOSITORY.git
```
enter directory
```bash
cd REPOSITORY
```

Create virtual environment.

```bash
python3 -m venv env
source env/bin/activate
```

Install dependencies.

```bash
pip install -r requirements.txt
pip install gunicorn
```

---

# 3. Django Production Settings(.env Recomended)

Edit `settings.py`.

```bash
cd project/settings.py
```

### Allowed Hosts

```python
ALLOWED_HOSTS = ["yourdomain.com", "www.yourdomain.com", "SERVER_IP"]
```

### Static Files*

```python
STATIC_URL = "/static/"
STATIC_ROOT = BASE_DIR / "static"
```

### Media Files*

```python
MEDIA_URL = "/media/"
MEDIA_ROOT = BASE_DIR / "media"
```

### Security Settings (Recommended)

```python
DEBUG = False

SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = "DENY"
```

---

# 4. Database Migration

```bash
python manage.py migrate
```

Create superuser:

```bash
python manage.py createsuperuser
```

Collect static files:

```bash
python manage.py collectstatic
```

Test Gunicorn:

```bash
gunicorn --bind 0.0.0.0:8000 projectname.wsgi:application
```

---

# 5. Gunicorn Systemd Service

Create service file.

```bash
sudo nano /etc/systemd/system/projectname.service
```

Example:

```
[Unit]
Description=Gunicorn daemon For project
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/event2

ExecStart=/var/www/event2/env/bin/gunicorn \
    --workers 3 \
    --bind unix:/run/event2/gunicorn.sock \
event-management.wsgi:application

RuntimeDirectory=event2
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target]
```

Reload systemd.

```bash
sudo systemctl daemon-reload
```

Start service.

```bash
sudo systemctl start project
```

Enable auto start.

```bash
sudo systemctl enable project
```

Check status.

```bash
sudo systemctl status project
```

---

# 6. Nginx Configuration

Create config file.

```bash
sudo nano /etc/nginx/sites-available/project
```

Example configuration:

```
server {
    listen 80;
    server_name project.com www.project.com;

    location /static/ {
        alias /var/www/project/static/;
    }

    location /media/ {
        alias /var/www/project/media/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/project/gunicorn.sock;
    }
}
```

Enable site.

```bash
sudo ln -s /etc/nginx/sites-available/project /etc/nginx/sites-enabled
```

Test configuration.

```bash
sudo nginx -t
```

Restart Nginx.

```bash
sudo systemctl restart nginx
```

---

# 7. Firewall (Optional)

Allow HTTP traffic.

```bash
sudo ufw allow 'Nginx Full'
```

---

# 8. HTTPS with Certbot

Install certbot.

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Run SSL setup.

```bash
sudo certbot --nginx -d project.com
```

Auto renewal test:

```bash
sudo certbot renew --dry-run
```
# Django Deployment: Folder Permissions Setup

When deploying Django in production under `/var/www/`, proper permissions are required so that **Gunicorn** and **Nginx** (running as `www-data` user) can access project files.

---

## 1. Change Ownership

```bash
sudo chown -R www-data:www-data /var/www/project
```
## 2. Set General Permissions
```bash
sudo chmod -R 755 /var/www/project
```
Purpose: Grants read/execute permission to everyone, but write permission only to the owner (www-data).

Why: Ensures files are accessible while keeping write access restricted.

---
3. Static & Media Folder Permissions

```bash
sudo chmod -R 775 /var/www/project/static
sudo chmod -R 775 /var/www/project/media
```


---

# 9. Performance Best Practices

### Gunicorn Workers

Recommended formula:

```
workers = (2 × CPU cores) + 1
```

Example for 2 CPU:

```
workers = 5
```

### Timeout

```
--timeout 120
```

### Log Files

```
--access-logfile -
--error-logfile -
```



---

# 10. Monitoring & Logs

Gunicorn logs:

```bash
sudo journalctl -u gunicorn -f
```

Nginx logs:

```bash
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log
```

---

# 11. Deployment Workflow

Typical production update workflow.

```bash
git pull origin main

source venv/bin/activate

pip install -r requirements.txt

python manage.py migrate

python manage.py collectstatic --noinput

sudo systemctl restart gunicorn
```

---

# 12. Recommended Project Structure

```
project/
│
├── project/
│   ├── settings.py
│   ├── urls.py
│   ├── wsgi.py
│
├── apps/
├── static/
├── media/
├── staticfiles/
├── venv/
├── manage.py
└── requirements.txt
```

---

# 13. Security Best Practices

• Never commit `.env` files
• Use environment variables for secrets
• Disable `DEBUG` in production
• Use HTTPS
• Restrict server SSH access
• Keep server packages updated

---

# 14. Useful Commands

Restart services:

```bash
sudo systemctl restart project
sudo systemctl restart nginx
```

Check open ports:

```bash
sudo ss -tulpn
```

Check disk usage:

```bash
df -h
```

---

# Summary

Production stack:

```
Client → Nginx → Gunicorn → Django
```

Nginx handles static files and reverse proxy while Gunicorn runs the Django application.

This setup provides a stable and scalable production environment for Django applications.
