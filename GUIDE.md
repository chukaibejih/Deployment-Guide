# Deployment Guide

This guide provides a step-by-step process to deploy a Django application on a DigitalOcean VPS using Gunicorn, Nginx, Redis, Celery, and PostgreSQL. It is designed to be project-agnostic, making it a reusable reference for any Django-based project.

## Step 1: Set Up Your DigitalOcean VPS

1. **Create a Droplet** on DigitalOcean with Ubuntu as the OS.
2. **Connect via SSH** to your VPS:
   ```sh
   ssh root@your_server_ip
   ```
3. **Create a new user** for security:
   ```sh
   adduser deployer
   usermod -aG sudo deployer
   ```
4. **Switch to the new user**:
   ```sh
   su - deployer
   ```

> **Why?** Running the application as the root user is a security risk. A separate user limits potential damage.

## Step 2: Install System Dependencies

```sh
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx curl redis-server -y
```

> **Why?** This installs required packages, including PostgreSQL, Redis, Nginx, and system dependencies.

## Step 3: Configure PostgreSQL

1. **Login to PostgreSQL:**
   ```sh
   sudo -i -u postgres
   psql
   ```
2. **Create a Database & User:**
   ```sql
   CREATE DATABASE mydatabase;
   CREATE USER myuser WITH PASSWORD 'mypassword';
   ALTER ROLE myuser SET client_encoding TO 'utf8';
   ALTER ROLE myuser SET default_transaction_isolation TO 'read committed';
   ALTER ROLE myuser SET timezone TO 'UTC';
   GRANT ALL PRIVILEGES ON DATABASE mydatabase TO myuser;
   ```
3. **Exit PostgreSQL:**
   ```sh
   \q
   exit
   ```

> **Why?** PostgreSQL is a robust database choice for Django applications, offering strong ACID compliance.

## Step 4: Clone Your Project from GitHub

1. **Navigate to the home directory:**
   ```sh
   cd /home/deployer
   ```
2. **Clone your project:**
   ```sh
   git clone https://github.com/yourusername/yourproject.git
   cd yourproject
   ```
3. **Create a virtual environment:**
   ```sh
   python3 -m venv venv
   source venv/bin/activate
   ```
4. **Install dependencies:**
   ```sh
   pip install -r requirements.txt
   ```

> **Why?** Using a virtual environment keeps dependencies isolated, avoiding conflicts with system-wide packages.

## Step 5: Configure Django Settings

1. **Set environment variables** (use `.env` or export variables directly):
   ```sh
   export SECRET_KEY='your_secret_key'
   export DATABASE_URL='postgres://myuser:mypassword@localhost/mydatabase'
   ```
2. **Run migrations:**
   ```sh
   python manage.py migrate
   ```
3. **Create a superuser (if needed):**
   ```sh
   python manage.py createsuperuser
   ```

> **Why?** Environment variables keep sensitive information out of version control.

## Step 6: Set Up Gunicorn

1. **Install Gunicorn:**
   ```sh
   pip install gunicorn
   ```
2. **Test Gunicorn:**
   ```sh
   gunicorn --bind 0.0.0.0:8000 yourproject.wsgi
   ```

> **Why?** Gunicorn serves as a production-grade WSGI server for Django applications.

## Step 7: Set Up Nginx

1. **Create an Nginx config file:**
   ```sh
   sudo nano /etc/nginx/sites-available/yourproject
   ```
2. **Add the following configuration:**
   ```nginx
   server {
       listen 80;
       server_name yourdomain.com;
       location / {
           proxy_pass http://127.0.0.1:8000;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```
3. **Enable the configuration and restart Nginx:**
   ```sh
   sudo ln -s /etc/nginx/sites-available/yourproject /etc/nginx/sites-enabled
   sudo nginx -t
   sudo systemctl restart nginx
   ```

> **Why?** Nginx serves as a reverse proxy, forwarding requests to Gunicorn.

## Step 8: Set Up Redis & Celery

1. **Enable Redis:**
   ```sh
   sudo systemctl enable redis
   sudo systemctl start redis
   ```
2. **Configure Celery in Django settings:**
   ```python
   CELERY_BROKER_URL = 'redis://localhost:6379/0'
   ```
3. **Start a Celery worker:**
   ```sh
   celery -A yourproject worker --loglevel=info
   ```

> **Why?** Celery handles background tasks, and Redis is used as a message broker.

## Step 9: Set Up a Domain Name

1. **Purchase a domain from Namecheap or any provider.**
2. **Update DNS records to point to your VPS IP.**
3. **Secure your Nginx server with Let’s Encrypt:**
   ```sh
   sudo apt install certbot python3-certbot-nginx
   sudo certbot --nginx -d yourdomain.com
   ```

> **Why?** A domain name makes your application accessible via a friendly URL.

## Step 10: Set Up Supervisor for Process Management

1. **Install Supervisor:**
   ```sh
   sudo apt install supervisor
   ```
2. **Create a config for Gunicorn:**
   ```sh
   sudo nano /etc/supervisor/conf.d/gunicorn.conf
   ```
   ```ini
   [program:gunicorn]
   command=/home/deployer/yourproject/venv/bin/gunicorn --workers 3 --bind unix:/home/deployer/yourproject/gunicorn.sock yourproject.wsgi:application
   directory=/home/deployer/yourproject
   user=deployer
   autostart=true
   autorestart=true
   stderr_logfile=/var/log/gunicorn.err.log
   stdout_logfile=/var/log/gunicorn.out.log
   ```
3. **Reload Supervisor:**
   ```sh
   sudo supervisorctl reread
   sudo supervisorctl update
   sudo supervisorctl start gunicorn
   ```

> **Why?** Supervisor keeps Gunicorn running in the background.

## Conclusion

This guide ensures a stable, scalable, and secure deployment of your Django application on a DigitalOcean VPS. As your application grows, consider setting up auto-scaling, monitoring, and logging solutions.

