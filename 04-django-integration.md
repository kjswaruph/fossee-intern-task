# Django Project Setup & SSO

## 1. Create Database

```bash
sudo mysql -u root -p
CREATE DATABASE djangodb;
CREATE USER 'djangouser'@'localhost' IDENTIFIED BY 'your_secure_password';
GRANT ALL PRIVILEGES ON djangodb.* TO 'djangouser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## 2. Set Up Django Project with Gunicorn

Install python3-devel mariadb-devel and gcc for installation of mysqlclient

```bash
# For mysqlclient
sudo dnf install python3-devel mariadb-devel gcc -y
```

```bash
# Create project directory and virtual environment
sudo mkdir /var/www/django_project
sudo chown your_username:your_username /var/www/django_project
cd /var/www/django_project
python3 -m venv venv
source venv/bin/activate

pip install django gunicorn mozilla-django-oidc mysqlclient
```

Create project

```bash
django-admin startproject mysite .
```

Configure network and database in settings.py

```python
ALLOWED_HOSTS = [ 'your_django_domain', 'your_droplet_ip', 'localhost']

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'djangodb',
        'USER': 'djangouser',
        'PASSWORD': 'your_secure_password',
        'HOST': 'localhost',
        'PORT': '3306'
    }
}

STATIC_URL = 'static/'

# Add this at the end
import os
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')

```

Finish

```bash
python manage.py collectstatic
python manage.py migrate
python manage.py createsuperuser
deactivate
```

## 3. Configure Apache as a Reverse Proxy:

Create `/etc/httpd/conf.d/django.conf`:

```conf
<VirtualHost *:80>
    ServerName your_django_domain
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
</VirtualHost>
```

Configure https

```bash
sudo certbot --apache -d your_django_domain
```

Certbot will generate /etc/httpd/conf.d/django-le-ssl.conf

Edit /etc/httpd/conf.d/django-le-ssl.conf and this line `RequestHeader set X-Forwarded-Proto "https"`

```apache
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName your_django_domain

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/

	RequestHeader set X-Forwarded-Proto "https"

    ErrorLog /var/log/httpd/django_error.log
    CustomLog /var/log/httpd/django_access.log combined

    SSLCertificateFile /etc/letsencrypt/live/your_django_domain/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/your_django_domain/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
</VirtualHost>
</IfModule>
```

Update /var/www/django_project/mysite/settings.py

```python
# Enfore HTTPS
SECURE_SSL_REDIRECT = True
CSRF_COOKIE_SECURE = True
SESSION_COOKIE_SECURE = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
```

## 4. Create Gunicorn service

```conf
[Unit]
Description=Gunicorn for Django mysite
After=network.target

[Service]
User=your_username
Group=apache
WorkingDirectory=/var/www/django_project
Environment="PATH=/var/www/django_project/venv/bin"
ExecStart=python3 /var/www/django_project/venv/bin/gunicorn --workers 3 --bind 127.0.0.1:8000 mysite.wsgi:application
Restart=always

[Install]
WantedBy=multi-user.target
```

Reload and start the gunicorn service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gunicorn
sudo systemctl status gunicorn
sudo systemctl restart httpd
```

![Django Installation](./screenshots/04-images/django-web.png)

## 5. Integrate Keycloak SSO

A. Create django client

- Open keycloak admin console and go to Manage realms and switch to sso-apps realm
- Manage > Clients > Create client
- Client ID: django
- Client Authentication: on
- Valid redirect URIs: https://your_django_domain/oidc/callback/
- Save and copy the Client Secret from the credentials tab

![django client configuration](./screenshots/04-images/django-client-1.png)

B. Update mysite/settings.py

```python
INSTALLED_APPS += ['mozilla_django_oidc']

AUTHENTICATION_BACKENDS = [
    'mozilla_django_oidc.auth.OIDCAuthenticationBackend',
    'django.contrib.auth.backends.ModelBackend',
]

OIDC_RP_CLIENT_ID = "django"
OIDC_RP_CLIENT_SECRET = "your_client_secret"

OIDC_OP_AUTHORIZATION_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/auth"
OIDC_OP_TOKEN_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-aps/protocol/openid-connect/token"
OIDC_OP_USER_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/userinfo"
OIDC_RP_SIGN_ALGO = "RS256"
OIDC_OP_JWKS_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/certs"
OIDC_OP_LOGOUT_ENDPOINT = "https://{your_keycloak_domain}/realms/sso-apps/protocol/openid-connect/logout"
OIDC_STORE_ID_TOKEN = True

LOGIN_URL = 'oidc_authentication_init'
LOGIN_REDIRECT_URL = '/profile/'
LOGOUT_REDIRECT_URL = '/'

```

## 6. Create Django app for homepage

```bash
cd /var/www/django_project
source venv/bin/activate
python manage.py startapp home
```

A new dir home will be created in django_project dir

Open mysite/settings.py and add home app in INSTALLED_APPS

```python
INSTALLED_APPS = [
	# ..Existing apps..
    'home',
]
```

Add view home/views.py

```python
from django.shortcuts import render, redirect
from django.contrib.auth.decorators import login_required
from django.contrib.auth import logout as django_logout
from django.conf import settings
import urllib.parse

# Create your views here.
def login(request):
    return render(request, 'login.html')

@login_required
def profile(request):
    return render(request, 'profile.html')

@login_required
def logout(request):

    django_logout(request)
    id_token = request.session.get('oidc_id_token')
    post_logout_redirect = request.build_absolute_uri(settings.LOGOUT_REDIRECT_URL)

    if id_token:
        logout_url = (
            f"{settings.OIDC_OP_LOGOUT_ENDPOINT}"
            f"?id_token_hint={id_token}"
            f"&post_logout_redirect_uri={urllib.parse.quote(post_logout_redirect)}"
        )
    else:
        logout_url = post_logout_redirect
    return redirect(logout_url)

```

Create home/templates/profile.html

```html
<!DOCTYPE html>
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Django Profile</title>
  </head>
  <body>
    <h1>Successful SSO login</h1>
    <p>Welcome, {{ user.get_username }} ({{ user.email }})</p>
    <form action="{% url 'logout' %}" method="post">
      {% csrf_token %}
      <button type="submit">Logout</button>
    </form>
  </body>
</html>
```

Create home/templates/login.html

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Django Login</title>
  </head>
  <body>
    <h1>Login Page</h1>
    <a href="{% url 'oidc_authentication_init' %}">Login with SSO</a>
  </body>
</html>
```

Add home/urls.py

```python
from django.urls import path
from .views import login, profile, logout

urlpatterns = [
    path('', login, name='login'),
    path('profile/', profile, name='profile'),
    path('logout/', logout, name='logout'),

]
```

Add home/urls.py in mysite/urls.py

```python
"""
URL configuration for mysite project.

The `urlpatterns` list routes URLs to views. For more information please see:
    https://docs.djangoproject.com/en/5.2/topics/http/urls/
Examples:
Function views
    1. Add an import:  from my_app import views
    2. Add a URL to urlpatterns:  path('', views.home, name='home')
Class-based views
    3. Add an import:  from other_app.views import Home
    4. Add a URL to urlpatterns:  path('', Home.as_view(), name='home')
Including another URLconf
    5. Import the include() function: from django.urls import include, path
    6. Add a URL to urlpatterns:  path('blog/', include('blog.urls'))
"""
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('oidc/', include('mozilla_django_oidc.urls')),
    path('', include('home.urls')),
]
```

Restart gunicorn service

```bash
sudo systemctl restart gunicorn
sudo systemctl restart httpd
```

## 7. Perform SSO login

- Now visiting `https://your_django_domain` you will first get login page, Click on Login with SSO
  ![Login page](./screenshots/04-images/login.png)
- You will be redirected to keycloak , sign in with sso-admin user
  ![](./screenshots/04-images/keycloak-login.png)
- After sign in, you will be redirected back to django login page
  ![SSO Login](./screenshots/04-images/sso.png)

**Next Steps:** Proceed with PHP App setup and configuration as documented in [05-php-integration.md](05-php-integration.md).
