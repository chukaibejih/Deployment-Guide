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
sudo -u postgres psql
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

## **4. Add SSH Key to GitHub**
To enable secure cloning, add your SSH key to GitHub:

1. Generate a new SSH key (if you haven't already):

   ```bash
   ssh-keygen -t ed25519 -C "your-email@example.com"
   ```
   Press **Enter** to save it in the default location and set a passphrase if desired.

2. Start the SSH agent and add your key:

   ```bash
   eval "$(ssh-agent -s)"
   ssh-add ~/.ssh/id_ed25519
   ```

3. Copy the SSH key to GitHub:

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
   - Go to **GitHub** → **Settings** → **SSH and GPG keys** → **New SSH key**.
   - Paste the key and save.

4. Test the connection:

   ```bash
   ssh -T git@github.com
   ```

   If successful, you’ll see:
   ```
   Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.
   ```

> **Why?** Using SSH ensures secure authentication without needing to enter passwords frequently.

---

## **5. Clone Your Project from GitHub**
Navigate to your home directory and clone the repository:

```bash
cd /home
git clone git@github.com:your_username/xtridelink.git
cd xtridelink
```

> **Why?** Cloning from GitHub ensures your project is easily deployable.

---

## **6. Set Up a Virtual Environment**

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

> **Why?** A virtual environment isolates dependencies and prevents conflicts.

---

## **7. Configure Django Settings**
Create and `.env` file:
```bash
sudo nano .env
```
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

### **Troubleshooting: Permission Denied Error**
If you encounter a `permission denied for schema public` error when running migrations, grant the necessary privileges to your database user by following these steps:

1. **Login to PostgreSQL as a Superuser**
   ```bash
   sudo -u postgres psql
   ```

2. **Switch to the correct database and grant privileges**
   ```sql
   \c xtridelink
   
   ALTER ROLE xtridelink_user WITH CREATEDB;
   GRANT ALL PRIVILEGES ON DATABASE xtridelink TO xtridelink_user;
   GRANT ALL PRIVILEGES ON SCHEMA public TO xtridelink_user;
   ```

3. **Exit PostgreSQL and retry migration**
   ```sql
   \q
   ```
   ```bash
   python manage.py migrate
   ```

This should resolve the permission issue and allow Django to apply migrations successfully.

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

## **8. Set Up Gunicorn**
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

## **9. Set Up Nginx**
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

## **Conclusion**
Your Django application is now fully deployed with:
✅ PostgreSQL as the database  
✅ Gunicorn as the application server  
✅ Nginx as the reverse proxy  
✅ Redis and Celery for background tasks  
✅ A custom domain attached

Enjoy your production-ready deployment! 🚀

