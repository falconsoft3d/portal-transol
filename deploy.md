# Despliegue de Transol en producción (Ubuntu + Nginx + Gunicorn)

> Guía paso a paso para desplegar el portal **Transol** (Django 6.1) en un servidor Ubuntu con PostgreSQL, Gunicorn y Nginx.
>
> Dominio de producción: **transol.xyz** (y **www.transol.xyz**).

---

## 1. Dependencias del servidor

```bash
sudo apt-get update && sudo apt-get -y upgrade

# Python y herramientas de compilación
sudo apt-get -y install python3 python3-pip python3-venv python3-dev build-essential

# PostgreSQL
sudo apt-get -y install postgresql postgresql-contrib libpq-dev

# Nginx y Supervisor
sudo apt-get -y install nginx supervisor

# Pillow (imágenes)
sudo apt-get -y install libjpeg-dev zlib1g-dev libpng-dev

sudo systemctl enable supervisor && sudo systemctl start supervisor
```

---

## 2. Base de datos PostgreSQL

```bash
sudo -u postgres psql
```

```sql
CREATE USER u_transol WITH PASSWORD 'CAMBIA_ESTA_PASSWORD';
CREATE DATABASE transol_prod OWNER u_transol;
GRANT ALL PRIVILEGES ON DATABASE transol_prod TO u_transol;
\q
```

---

## 3. Usuario de la aplicación

```bash
sudo adduser transol
sudo gpasswd -a transol sudo
su - transol
```

---

## 4. Clonar el repositorio y entorno virtual

```bash
cd /home/transol
python3 -m venv venv
source venv/bin/activate

git clone https://github.com/falconsoft3d/portal-transol.git app
cd app

pip install -r requirements.txt
pip install gunicorn psycopg2-binary
```

---

## 5. Configuración de producción (settings.py)

Edita `config/settings.py` o usa variables de entorno. Cambios mínimos requeridos:

```python
DEBUG = False

ALLOWED_HOSTS = ['transol.xyz', 'www.transol.xyz', 'IP_DEL_SERVIDOR']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'transol_prod',
        'USER': 'u_transol',
        'PASSWORD': 'CAMBIA_ESTA_PASSWORD',
        'HOST': 'localhost',
        'PORT': '',
    }
}

STATIC_ROOT = '/home/transol/static/'
MEDIA_ROOT  = '/home/transol/media/'
MEDIA_URL   = '/media/'
STATIC_URL  = '/static/'

# Genera una nueva SECRET_KEY segura:
# python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
SECRET_KEY = 'GENERA_UNA_NUEVA_SECRET_KEY'
```

---

## 6. Preparar la app

```bash
python manage.py migrate
python manage.py collectstatic --noinput
python manage.py createsuperuser
```

---

## 7. Configurar Gunicorn

```bash
mkdir -p /home/transol/run /home/transol/logs
touch /home/transol/logs/gunicorn-error.log
```

Crea `/home/transol/venv/bin/gunicorn_start`:

```bash
vim /home/transol/venv/bin/gunicorn_start
```

```bash
#!/bin/bash

NAME="transol"
DIR=/home/transol/app
USER=transol
GROUP=transol
WORKERS=3
BIND=unix:/home/transol/run/gunicorn.sock
DJANGO_SETTINGS_MODULE=config.settings
DJANGO_WSGI_MODULE=config.wsgi
LOG_LEVEL=error

cd $DIR
source /home/transol/venv/bin/activate

export DJANGO_SETTINGS_MODULE=$DJANGO_SETTINGS_MODULE
export PYTHONPATH=$DIR:$PYTHONPATH

exec gunicorn ${DJANGO_WSGI_MODULE}:application \
  --name $NAME \
  --workers $WORKERS \
  --user=$USER \
  --group=$GROUP \
  --bind=$BIND \
  --log-level=$LOG_LEVEL \
  --log-file=-
```

```bash
chmod u+x /home/transol/venv/bin/gunicorn_start
```

---

## 8. Configurar Supervisor

```bash
sudo vim /etc/supervisor/conf.d/transol.conf
```

```ini
[program:transol]
command=/home/transol/venv/bin/gunicorn_start
user=transol
autostart=true
autorestart=true
redirect_stderr=true
stdout_logfile=/home/transol/logs/gunicorn-error.log
```

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl status transol
```

---

## 9. Configurar Nginx

```bash
sudo vim /etc/nginx/sites-available/transol
```

```nginx
upstream transol_app_server {
    server unix:/home/transol/run/gunicorn.sock fail_timeout=0;
}

server {
    listen 80;
    server_name transol.xyz www.transol.xyz;

    client_max_body_size 50M;

    access_log /home/transol/logs/nginx-access.log;
    error_log  /home/transol/logs/nginx-error.log;

    location /static/ {
        alias /home/transol/static/;
    }

    location /media/ {
        alias /home/transol/media/;
    }

    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_redirect   off;
        proxy_pass       http://transol_app_server;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/transol /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 10. SSL con Let's Encrypt (recomendado)

```bash
sudo apt-get install certbot python3-certbot-nginx
sudo certbot --nginx -d transol.xyz -d www.transol.xyz
```

---

## 11. Actualizar la app (deploy de nuevas versiones)

```bash
su - transol
cd /home/transol/app
source /home/transol/venv/bin/activate
git pull
pip install -r requirements.txt
python manage.py migrate
python manage.py collectstatic --noinput
sudo supervisorctl restart transol
```

---

## Permisos de carpetas media

```bash
sudo chown -R transol:transol /home/transol/media/
sudo chmod -R 755 /home/transol/media/
```

---

## Comandos útiles de Supervisor

```bash
sudo supervisorctl status
sudo supervisorctl stop transol
sudo supervisorctl start transol
sudo supervisorctl restart transol
```


