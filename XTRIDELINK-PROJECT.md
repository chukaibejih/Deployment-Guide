# Deploying a Django Application on DigitalOcean VPS using the XTRIDELINK project for a relatable understanding for me.

This guide provides a step-by-step process for deploying a Django application on a **DigitalOcean VPS** using **Gunicorn, Nginx, Redis, Celery, PostgreSQL**, and setting up a **custom domain**.

---

## **1. SSH into Your VPS**
Connect to your server using SSH:

```bash
ssh root@your_server_ip
```

If using a non-root user:

```bash
ssh your_user@your_server_ip
```

> **Note:** Using SSH allows secure remote access to your VPS.

---

## **2. Update & Install Dependencies**
Ensure your server is up-to-date and install required packages:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3-pip python3-dev libpq-dev postgresql postgresql-contrib nginx curl supervisor redis-server -y
```

> **Why?** This installs essential dependencies, including Python, PostgreSQL, Nginx, Redis, and Celery.

---

## **3. Set Up PostgreSQL Database**
Switch to the PostgreSQL user:

```bash
sudo -i -u postgres
```

Create a new database and user:

```sql
CREATE DATABASE xtridelink;
CREATE USER xtridelink_user WITH PASSWORD 'secure_password';
ALTER ROLE xtridelink_user SET client_encoding TO 'utf8';
ALTER ROLE xtridelink_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE xtridelink_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE xtridelink TO xtridelink_user;
```

Exit PostgreSQL:
```bash
\q
exit
```

> **Why?** PostgreSQL is a production-grade database that ensures data integrity and performance.

---

## **4. Clone Your Project from GitHub**
Navigate to your home directory and clone the repository:

```bash
cd /home
git clone git@github.com:your_username/xtridelink.git
cd xtridelink
```

> **Why?** Cloning from GitHub ensures your project is easily deployable.

---

## **5. Set Up a Virtual Environment**

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

> **Why?** A virtual environment isolates dependencies and prevents conflicts.

---

## **6. Configure Django Settings**
Edit your `.env` file with the correct values:

```plaintext
DEBUG=False
ALLOWED_HOSTS=your_server_ip, yourdomain.com
DATABASE_URL=postgres://xtridelink_user:secure_password@localhost/xtridelink
```

Apply migrations:

```bash
python manage.py migrate
```

Collect static files:

```bash
python manage.py collectstatic --noinput
```

Create a superuser:

```bash
python manage.py createsuperuser
```

> **Why?** These steps ensure your application is ready for production with proper database setup.

---

## **7. Set Up Gunicorn**
Install Gunicorn:

```bash
pip install gunicorn
```

Test Gunicorn:

```bash
gunicorn --bind 0.0.0.0:8000 config.wsgi:application
```

Stop Gunicorn (**Ctrl + C**).

Create a **Gunicorn systemd service**:

```bash
sudo nano /etc/systemd/system/xtridelink.service
```

Paste the following:

```ini
[Unit]
Description=Gunicorn daemon for xtridelink
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/home/xtridelink
ExecStart=/home/xtridelink/venv/bin/gunicorn --workers 3 --bind unix:/home/xtridelink/xtridelink.sock config.wsgi:application

[Install]
WantedBy=multi-user.target
```

Save and exit. Then start and enable Gunicorn:

```bash
sudo systemctl daemon-reload
sudo systemctl start xtridelink
sudo systemctl enable xtridelink
```

> **Why?** Gunicorn is a production-grade WSGI server for running Django applications.

---

## **8. Set Up Nginx**
Create an Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/xtridelink
```

Paste the following:

```nginx
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/xtridelink/xtridelink.sock;
    }

    location /static/ {
        root /home/xtridelink;
    }

    location /media/ {
        root /home/xtridelink;
    }
}
```

Enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/xtridelink /etc/nginx/sites-enabled
sudo nginx -t
sudo systemctl restart nginx
```

> **Why?** Nginx acts as a reverse proxy, handling HTTP requests efficiently.

---

## **9. Set Up Redis**
Enable and start Redis:

```bash
sudo systemctl enable redis-server
sudo systemctl start redis-server
```

Test Redis:

```bash
redis-cli ping
```

> **Why?** Redis is used for caching and message brokering with Celery.

---

## **10. Set Up Celery**
Install Celery:

```bash
pip install celery
```

Create a **Celery systemd service**:

```bash
sudo nano /etc/systemd/system/celery.service
```

Paste the following:

```ini
[Unit]
Description=Celery Service
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/home/xtridelink
ExecStart=/home/xtridelink/venv/bin/celery -A config worker --loglevel=info

[Install]
WantedBy=multi-user.target
```

Start and enable Celery:

```bash
sudo systemctl daemon-reload
sudo systemctl start celery
sudo systemctl enable celery
```

> **Why?** Celery handles background task execution.

---

## **11. Attach Domain to VPS**
1. Update **DNS A record** at your domain registrar:
   - **Host:** `@`
   - **Type:** `A`
   - **Value:** `your_server_ip`
2. Add a **CNAME record** for `www`:
   - **Host:** `www`
   - **Type:** `CNAME`
   - **Value:** `yourdomain.com`
3. Restart Nginx:

```bash
sudo systemctl restart nginx
```

> **Why?** Configuring DNS ensures your domain points to your VPS.

---

## **Conclusion**
Your Django application is now fully deployed with:
✅ PostgreSQL as the database  
✅ Gunicorn as the application server  
✅ Nginx as the reverse proxy  
✅ Redis and Celery for background tasks  
✅ A custom domain attached

Enjoy your production-ready deployment! 🚀




