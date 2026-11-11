# Django Production Setup: Nginx + Gunicorn

A simple reference guide for deploying a Django application on Ubuntu using:

- **Django** — application framework
- **Gunicorn** — WSGI application server
- **Nginx** — web server and reverse proxy
- **SQLite** — default database (no PostgreSQL required)

> These instructions are suitable as a learning/reference setup. For a public production VPS, use a Linux-native project directory (for example `/home/ubuntu/myproject`), configure HTTPS, keep secrets out of Git, and run Gunicorn as a systemd service.

---

## 1. Example project structure

This guide assumes the following structure:

```text
myproject/
├── config/
│   ├── __init__.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
├── env/                 # Python virtual environment
├── manage.py
└── requirements.txt
```

Here, `config` is the Django project package containing `wsgi.py`. Replace it with your actual package name if different.

## 2. Install required packages

```bash
sudo apt update
sudo apt install -y python3 python3-pip python3-venv nginx
```

## 3. Prepare the Django environment

Go to the project directory:

```bash
cd /home/ubuntu/myproject
```

Activate the virtual environment. If it does not exist yet:

```bash
python3 -m venv env
source env/bin/activate
```

Install project dependencies and Gunicorn:

```bash
pip install -r requirements.txt
pip install gunicorn
```

## 4. Configure Django

In `config/settings.py`, set the appropriate hostnames/IP addresses:

```python
DEBUG = False

ALLOWED_HOSTS = [
    "YOUR_SERVER_IP",
    "yourdomain.com",
    "www.yourdomain.com",
]
```

For local testing, you may use:

```python
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]
```

Keep Django's default SQLite configuration if you do not need PostgreSQL:

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.sqlite3",
        "NAME": BASE_DIR / "db.sqlite3",
    }
}
```

Do not commit production `SECRET_KEY`, passwords, or other secrets to Git.

## 5. Run migrations and collect static files

With the virtual environment active:

```bash
python manage.py migrate
python manage.py collectstatic
```

Create an admin user if needed:

```bash
python manage.py createsuperuser
```

Make sure `STATIC_ROOT` is configured in `settings.py`, for example:

```python
STATIC_ROOT = BASE_DIR / "staticfiles"
```

## 6. Test Gunicorn

Run this from the project directory:

```bash
gunicorn --bind 127.0.0.1:8000 config.wsgi:application
```

Open another terminal and test:

```bash
curl http://127.0.0.1:8000
```

Stop the foreground Gunicorn process with `Ctrl + C`.

## 7. Configure Nginx

Create an Nginx site configuration:

```bash
sudo nano /etc/nginx/sites-available/myproject
```

Add:

```nginx
server {
    listen 80;
    server_name YOUR_SERVER_IP yourdomain.com www.yourdomain.com;

    location /static/ {
        alias /home/ubuntu/myproject/staticfiles/;
    }

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Change the paths and `server_name` values to match your server and domain.

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/myproject /etc/nginx/sites-enabled/myproject
```

If the default Nginx site conflicts, disable it:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
```

Check the configuration and reload Nginx:

```bash
sudo nginx -t
sudo systemctl enable nginx
sudo systemctl restart nginx
```

## 8. Run Gunicorn with systemd

Create a service file:

```bash
sudo nano /etc/systemd/system/myproject.service
```

Example configuration (change `ubuntu` and paths to your actual Linux username/project path):

```ini
[Unit]
Description=Gunicorn for Django myproject
After=network.target

[Service]
User=ubuntu
Group=www-data
WorkingDirectory=/home/ubuntu/myproject
ExecStart=/home/ubuntu/myproject/env/bin/gunicorn \
    --workers 2 \
    --bind 127.0.0.1:8000 \
    config.wsgi:application
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable myproject
sudo systemctl start myproject
```

Check status and logs:

```bash
sudo systemctl status myproject
sudo journalctl -u myproject -f
```

Restart after deploying code changes:

```bash
sudo systemctl restart myproject
```

## 9. Open the website

- On the VPS: `http://YOUR_SERVER_IP`
- With a domain: `http://yourdomain.com`

For a public VPS, allow inbound TCP ports **80** and **443** in the VPS/cloud firewall. Do not expose Gunicorn port `8000` publicly; it should listen on `127.0.0.1`.

## 10. Enable HTTPS (recommended for public websites)

After your domain's DNS points to the VPS and port 80 is reachable:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Follow Certbot's prompts.

---

## Useful commands

| Task | Command |
|---|---|
| Check Nginx | `sudo systemctl status nginx` |
| Restart Nginx | `sudo systemctl restart nginx` |
| Test Nginx config | `sudo nginx -t` |
| Check Gunicorn service | `sudo systemctl status myproject` |
| Restart Gunicorn | `sudo systemctl restart myproject` |
| View Gunicorn logs | `sudo journalctl -u myproject -f` |
| Check listening ports | `sudo ss -ltnp` |

## Troubleshooting

**502 Bad Gateway**

Check whether Gunicorn is running:

```bash
sudo systemctl status myproject
sudo journalctl -u myproject -n 50 --no-pager
```

**Nginx configuration error**

```bash
sudo nginx -t
```

**Django Invalid HTTP_HOST / DisallowedHost**

Add the requested IP address or domain to `ALLOWED_HOSTS` in `settings.py`, then restart Gunicorn.

**Static files are missing**

Run:

```bash
python manage.py collectstatic
```

Confirm that Nginx's `/static/` alias points to the same directory as Django's `STATIC_ROOT`.

---

## Important notes

- SQLite is fine for a small/simple app, but it has write-concurrency limitations. Consider PostgreSQL if the application grows or has many simultaneous writes.
- Use a strong, private `SECRET_KEY` and environment variables for production secrets.
- Back up `db.sqlite3` and uploaded media regularly.
- Do not use Django's `runserver` for public production deployment.
- The commands above assume a normal Ubuntu VPS. WSL on Windows is useful for learning, but it is not the same as deploying to a public VPS.
