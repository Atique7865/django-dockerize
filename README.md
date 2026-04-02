# 🐍 Django Dockerized Production Setup

A complete guide to building, structuring, and deploying a Django application in production using Docker, Docker Compose, Nginx, and Gunicorn.

---

## 📁 Project Folder Structure

```
django-dockerize/
│
├── apps/
│   ├── core/                        # Core/shared utilities
│   │   ├── migrations/
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── tests.py
│   │
│   ├── contact/                     # Contact app (contact form, inquiries)
│   │   ├── migrations/
│   │   │   └── __init__.py
│   │   ├── __init__.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── forms.py
│   │   ├── models.py
│   │   ├── serializers.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   └── tests.py
│   │
│   └── order/                       # Order app (orders, payments)
│       ├── migrations/
│       │   └── __init__.py
│       ├── __init__.py
│       ├── admin.py
│       ├── apps.py
│       ├── forms.py
│       ├── models.py
│       ├── serializers.py
│       ├── views.py
│       ├── urls.py
│       └── tests.py
│
├── config/                          # Project configuration (replaces inner project folder)
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings/
│   │   ├── __init__.py
│   │   ├── base.py                  # Shared settings
│   │   ├── development.py           # Dev-only settings
│   │   └── production.py            # Production settings
│   ├── urls.py
│   └── wsgi.py
│
├── docker/
│   ├── nginx/
│   │   ├── nginx.conf               # Main Nginx config
│   │   └── default.conf             # Server block config
│   ├── django/
│   │   └── Dockerfile               # Django app Dockerfile
│   └── entrypoint.sh                # Entrypoint script
│
├── static/                          # Collected static files (git-ignored)
├── media/                           # User uploaded files (git-ignored)
├── templates/                       # Global HTML templates
│   ├── base.html
│   ├── contact/
│   │   └── contact.html
│   └── order/
│       └── order_list.html
│
├── requirements/
│   ├── base.txt                     # Shared dependencies
│   ├── development.txt              # Dev dependencies
│   └── production.txt               # Production dependencies
│
├── .env                             # Environment variables (git-ignored)
├── .env.example                     # Sample env file (committed)
├── .gitignore
├── docker-compose.yml               # Development compose
├── docker-compose.prod.yml          # Production compose
└── manage.py
```

---

## ⚙️ Step 1 — Create the Django Project

```bash
# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate          # On Windows: venv\Scripts\activate

# Install Django
pip install django djangorestframework gunicorn psycopg2-binary python-decouple

# Start project (using config as the settings folder)
django-admin startproject config .

# Create apps inside the apps/ folder
mkdir apps
python manage.py startapp core apps/core
python manage.py startapp contact apps/contact
python manage.py startapp order apps/order
```

---

## 📦 Step 2 — Requirements Files

### `requirements/base.txt`
```
Django>=4.2
djangorestframework>=3.14
psycopg2-binary>=2.9
gunicorn>=21.2
python-decouple>=3.8
Pillow>=10.0
whitenoise>=6.6
```

### `requirements/development.txt`
```
-r base.txt
django-debug-toolbar>=4.2
```

### `requirements/production.txt`
```
-r base.txt
sentry-sdk>=1.40
```

---

## ⚙️ Step 3 — Django Settings

### `config/settings/base.py`
```python
from pathlib import Path
from decouple import config

BASE_DIR = Path(__file__).resolve().parent.parent.parent

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='').split(',')

DJANGO_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]

THIRD_PARTY_APPS = [
    'rest_framework',
]

LOCAL_APPS = [
    'apps.core',
    'apps.contact',
    'apps.order',
]

INSTALLED_APPS = DJANGO_APPS + THIRD_PARTY_APPS + LOCAL_APPS

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',   # Serve static files
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'config.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

WSGI_APPLICATION = 'config.wsgi.application'

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('POSTGRES_DB'),
        'USER': config('POSTGRES_USER'),
        'PASSWORD': config('POSTGRES_PASSWORD'),
        'HOST': config('DB_HOST', default='db'),
        'PORT': config('DB_PORT', default='5432'),
    }
}

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'static'
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'

MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

### `config/settings/production.py`
```python
from .base import *

DEBUG = False

SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

---

## 🌍 Step 4 — Environment Variables

### `.env.example`
```env
SECRET_KEY=your-secret-key-here
DEBUG=False
ALLOWED_HOSTS=localhost,127.0.0.1,yourdomain.com

POSTGRES_DB=mydb
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword
DB_HOST=db
DB_PORT=5432
```

Copy and fill in your actual values:
```bash
cp .env.example .env
```

---

## 🐳 Step 5 — Dockerfile (Django)

### `docker/django/Dockerfile`
```dockerfile
# Pull official base image
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements/ requirements/
RUN pip install --upgrade pip && pip install -r requirements/production.txt

# Copy project
COPY . .

# Collect static files
RUN python manage.py collectstatic --noinput --settings=config.settings.production

# Run entrypoint script
COPY docker/entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

EXPOSE 8000

ENTRYPOINT ["/entrypoint.sh"]
```

---

## 📝 Step 6 — Entrypoint Script

### `docker/entrypoint.sh`
```bash
#!/bin/sh

echo "Waiting for PostgreSQL..."
while ! nc -z $DB_HOST $DB_PORT; do
  sleep 0.1
done
echo "PostgreSQL started"

echo "Running migrations..."
python manage.py migrate --settings=config.settings.production

echo "Starting Gunicorn..."
exec gunicorn config.wsgi:application \
    --bind 0.0.0.0:8000 \
    --workers 3 \
    --timeout 120 \
    --access-logfile - \
    --error-logfile -
```

> Install `netcat` in the Dockerfile if not present:
> ```dockerfile
> RUN apt-get install -y gcc libpq-dev netcat-openbsd
> ```

---

## 🌐 Step 7 — Nginx Configuration

### `docker/nginx/nginx.conf`
```nginx
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile        on;
    keepalive_timeout 65;
    client_max_body_size 20M;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml;

    include /etc/nginx/conf.d/*.conf;
}
```

### `docker/nginx/default.conf`
```nginx
upstream django {
    server web:8000;
}

server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location / {
        proxy_pass         http://django;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_read_timeout 90;
    }

    location /static/ {
        alias /app/static/;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    location /media/ {
        alias /app/media/;
        expires 7d;
    }
}
```

> **Note:** Place your SSL certificate files (`fullchain.pem`, `privkey.pem`) inside a `docker/nginx/ssl/` directory or use Let's Encrypt with Certbot.

---

## 🐳 Step 8 — Docker Compose Files

### `docker-compose.yml` (Development)
```yaml
version: '3.9'

services:
  db:
    image: postgres:15-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "5432:5432"

  web:
    build:
      context: .
      dockerfile: docker/django/Dockerfile
    command: python manage.py runserver 0.0.0.0:8000 --settings=config.settings.development
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    env_file:
      - .env
    depends_on:
      - db

volumes:
  postgres_data:
```

### `docker-compose.prod.yml` (Production)
```yaml
version: '3.9'

services:
  db:
    image: postgres:15-alpine
    restart: always
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

  web:
    build:
      context: .
      dockerfile: docker/django/Dockerfile
    restart: always
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    env_file:
      - .env
    depends_on:
      - db
    expose:
      - 8000

  nginx:
    image: nginx:1.25-alpine
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./docker/nginx/ssl:/etc/nginx/ssl
      - static_volume:/app/static
      - media_volume:/app/media
    depends_on:
      - web

volumes:
  postgres_data:
  static_volume:
  media_volume:
```

---

## 🚀 Step 9 — Build & Run

### Development
```bash
# Build and start all services
docker-compose up --build

# Run in detached mode
docker-compose up -d --build

# View logs
docker-compose logs -f web

# Stop services
docker-compose down
```

### Production
```bash
# Build and start production services
docker-compose -f docker-compose.prod.yml up -d --build

# Apply migrations manually (if entrypoint doesn't run)
docker-compose -f docker-compose.prod.yml exec web python manage.py migrate

# Create superuser
docker-compose -f docker-compose.prod.yml exec web python manage.py createsuperuser

# View logs
docker-compose -f docker-compose.prod.yml logs -f

# Restart a single service
docker-compose -f docker-compose.prod.yml restart nginx

# Stop and remove containers
docker-compose -f docker-compose.prod.yml down
```

---

## 🗃️ Step 10 — Sample App: Contact

### `apps/contact/models.py`
```python
from django.db import models

class Contact(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField()
    subject = models.CharField(max_length=200)
    message = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"{self.name} - {self.subject}"
```

### `apps/contact/serializers.py`
```python
from rest_framework import serializers
from .models import Contact

class ContactSerializer(serializers.ModelSerializer):
    class Meta:
        model = Contact
        fields = '__all__'
```

### `apps/contact/views.py`
```python
from rest_framework import generics
from .models import Contact
from .serializers import ContactSerializer

class ContactCreateView(generics.CreateAPIView):
    queryset = Contact.objects.all()
    serializer_class = ContactSerializer
```

### `apps/contact/urls.py`
```python
from django.urls import path
from .views import ContactCreateView

urlpatterns = [
    path('contact/', ContactCreateView.as_view(), name='contact-create'),
]
```

---

## 🛒 Step 11 — Sample App: Order

### `apps/order/models.py`
```python
from django.db import models
from django.contrib.auth.models import User

class Order(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('processing', 'Processing'),
        ('shipped', 'Shipped'),
        ('delivered', 'Delivered'),
        ('cancelled', 'Cancelled'),
    ]

    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name='orders')
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='pending')
    total_price = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"Order #{self.id} by {self.user.username}"


class OrderItem(models.Model):
    order = models.ForeignKey(Order, on_delete=models.CASCADE, related_name='items')
    product_name = models.CharField(max_length=200)
    quantity = models.PositiveIntegerField(default=1)
    price = models.DecimalField(max_digits=10, decimal_places=2)

    def __str__(self):
        return f"{self.product_name} x{self.quantity}"
```

### `apps/order/serializers.py`
```python
from rest_framework import serializers
from .models import Order, OrderItem

class OrderItemSerializer(serializers.ModelSerializer):
    class Meta:
        model = OrderItem
        fields = ['id', 'product_name', 'quantity', 'price']

class OrderSerializer(serializers.ModelSerializer):
    items = OrderItemSerializer(many=True, read_only=True)

    class Meta:
        model = Order
        fields = ['id', 'user', 'status', 'total_price', 'items', 'created_at']
```

### `apps/order/views.py`
```python
from rest_framework import generics, permissions
from .models import Order
from .serializers import OrderSerializer

class OrderListCreateView(generics.ListCreateAPIView):
    serializer_class = OrderSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Order.objects.filter(user=self.request.user)

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)

class OrderDetailView(generics.RetrieveUpdateDestroyAPIView):
    serializer_class = OrderSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        return Order.objects.filter(user=self.request.user)
```

### `apps/order/urls.py`
```python
from django.urls import path
from .views import OrderListCreateView, OrderDetailView

urlpatterns = [
    path('orders/', OrderListCreateView.as_view(), name='order-list'),
    path('orders/<int:pk>/', OrderDetailView.as_view(), name='order-detail'),
]
```

---

## 🔗 Step 12 — Root URL Configuration

### `config/urls.py`
```python
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('apps.contact.urls')),
    path('api/', include('apps.order.urls')),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

## 🔒 Step 13 — SSL with Let's Encrypt (Optional but Recommended)

If you don't have SSL certificates, use Certbot:

```bash
# On your server, install Certbot
sudo apt install certbot

# Generate certificate (stop Nginx first if running on port 80)
sudo certbot certonly --standalone -d yourdomain.com -d www.yourdomain.com

# Certificates will be at:
# /etc/letsencrypt/live/yourdomain.com/fullchain.pem
# /etc/letsencrypt/live/yourdomain.com/privkey.pem

# Copy or volume-mount them into the container
```

Update `docker-compose.prod.yml` nginx volumes:
```yaml
- /etc/letsencrypt/live/yourdomain.com/fullchain.pem:/etc/nginx/ssl/fullchain.pem
- /etc/letsencrypt/live/yourdomain.com/privkey.pem:/etc/nginx/ssl/privkey.pem
```

---

## 🧹 Step 14 — .gitignore

```gitignore
# Python
__pycache__/
*.pyc
*.pyo
*.pyd
*.env
venv/
.venv/

# Django
static/
media/
*.log

# Docker
.env

# IDE
.vscode/
.idea/
*.swp
```

---

## ✅ Production Checklist

| Task | Status |
|------|--------|
| `DEBUG=False` in production settings | ✅ |
| Strong `SECRET_KEY` set in `.env` | ✅ |
| Database credentials secured in `.env` | ✅ |
| Static files collected via `collectstatic` | ✅ |
| HTTPS enabled via Nginx + SSL | ✅ |
| `SECURE_SSL_REDIRECT=True` | ✅ |
| `SESSION_COOKIE_SECURE=True` | ✅ |
| Gunicorn with multiple workers | ✅ |
| Nginx as reverse proxy | ✅ |
| PostgreSQL as production DB | ✅ |
| Health check / restart policies | ✅ |

---

## 🏗️ Architecture Overview

```
Browser
   │
   ▼
[Nginx :443]  ◄──── SSL Termination + Static Files
   │
   ▼
[Gunicorn :8000]  ◄──── Django WSGI App
   │
   ▼
[PostgreSQL :5432]  ◄──── Persistent Database
```

---

## 📌 Useful Commands

```bash
# Shell into Django container
docker-compose -f docker-compose.prod.yml exec web bash

# Run Django management commands
docker-compose -f docker-compose.prod.yml exec web python manage.py <command>

# View database
docker-compose -f docker-compose.prod.yml exec db psql -U myuser -d mydb

# Rebuild single service
docker-compose -f docker-compose.prod.yml up -d --build web

# Prune unused Docker resources
docker system prune -f
```

---

## 📚 Tech Stack

| Component | Technology |
|-----------|-----------|
| Web Framework | Django 4.2+ |
| API | Django REST Framework |
| WSGI Server | Gunicorn |
| Reverse Proxy | Nginx |
| Database | PostgreSQL 15 |
| Containerization | Docker + Docker Compose |
| Static Files | WhiteNoise |
| Environment | python-decouple |
