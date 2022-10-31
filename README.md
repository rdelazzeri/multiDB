# Django MultiDB
# Django multiple database project
Projeto para uma aplicação com multiplos clientes em bancos de dados separados com um único código.

## Sequencia inicial:

Criar ambiente
```
python -m venv venv 
.\venv\scripts\activate
```

Instalar dependências do projeto
```
python -m pip install --upgrade pip
pip install django
```

Criar o projeto
```
django-admin startproject multidb
python manage.py startapp core

```

Fazendo alterações no `settings.py`
```
...

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'core', #novo
]

```

Configurando o banco de dados

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'multidb.sqlite3',
    }
}
```

Criando o banco de dados e o superuser
```
python manage.py migrate
python manage.py cratesuperuser
```

Neste ponto o sistema deve estar funcionando de forma tradicional.

## Adicionando multiplos bancos de dados
Agora serão incluídos os bancos de dados das aplicações cliente. No arquivo `settings.py` acrescente um database para cada cliente.
```
ENGINE = 'django.db.backends.sqlite3'

DATABASES = {
    'default': {'ENGINE': ENGINE, 'NAME': BASE_DIR / 'multidb.sqlite3'},
    'multidb1': {'ENGINE': ENGINE, 'NAME': BASE_DIR / 'multidb1.sqlite3'},
    'multidb2': {'ENGINE': ENGINE, 'NAME': BASE_DIR / 'multidb2.sqlite3'},
}

``` 

Adicionar um arquivo `utils.py` dentro da pasta `core` com o seguinte conteúdo:

```
from django.db import connection

def hostname_from_the_request(request):
    return request.get_host().split(":")[0].lower()

def tenant_db_from_the_request(request):
    hostname = hostname_from_the_request(request)
    tenants_map = get_tenants_map()
    return tenants_map.get(hostname)

def get_tenants_map():
    return {
        "nairobi.school.local": "nairobi",
        "accra.school.local": "accra"
    }
```

Adicionar um arquivo `middleware.py` na pasta `core` com o seguinte conteúdo:
```
import threading
from django.db import connections
from .utils import tenant_db_from_the_request

Thread_Local = threading.local()

class MultiDbMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        db = tenant_db_from_the_request(request)
        setattr(Thread_Local, "DB", db)
        response = self.get_response(request)
        return response

def get_current_db_name():
    return getattr(Thread_Local, "DB", None)

def set_db_for_router(db):
    setattr(Thread_Local, "DB", db)
```

Adicionar um arquivo `router.py` na pasta `core` com o seguinte conteúdo:
```
from .middleware import get_current_db_name

class MultiDbRouter:
    def db_for_read(self, model, **hints):
        return get_current_db_name()

    def db_for_write(self, model, **hints):
        return get_current_db_name()

    def allow_relation(self, *args, **kwargs):
        return True

    def allow_syncdb(self, *args, **kwargs):
        return None

    def allow_migrate(self, *args, **kwargs):
        return None
```

## Registrar o middleware e o router me `settings.py`
```
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'core.middleware.MultiDbMiddleware', # new
]
DATABASE_ROUTERS = ['core.router.MultiDbRouter']  # new
```

### Configurando os host names

Now, we need to map the hostnames to the local machine.

For Linux users, navigate to the /etc/hosts and for Windows users, follow the path C:\Windows\System32\Drivers\etc\.

Open the host file using notepad or any other text editor and add our hosts as shown below:

127.0.0.1 multidb.local
127.0.0.1 multidb1.local
127.0.0.1 multidb2.local


We also need to update the ALLOWED_HOSTS in our settings.py to:
```
ALLOWED_HOSTS = ['multidb.local', '.multidb1.local', '.multidb2.local']
``` 

Precisamos criar agora um custom manage.py
No nível da pasta de projeto, criar um arquivo `multidb_manage.py` com este conteúdo:
```
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys

from School.middleware import set_db_for_router #new

if __name__ == "__main__":                  
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'multitenant.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc

    #new 
    from django.db import connection

    args = sys.argv
    db = args[1]
    with connection.cursor() as cursor:
        set_db_for_router(db)
        del args[1]
        execute_from_command_line(args) 
```


Fazer as migrações:

```
python manage.py migrate --database=multidb1
python mange.py createsuperuser --database=multidb1

...

python manage.py migrate --database=multidb2
python mange.py createsuperuser --database=multidb2
```

Neste momento o projeto já deve estar funcionando. Faça o teste

```
python manage.py runserver
```

para acessar os sites, agora será preciso buscar os hosts de cada cliente:
http://multidb.local ou http://multidb1.local ou http://multidb2.local


Referencias:

https://www.section.io/engineering-education/implement-multitenancy-with-multiple-databases-in-django/

https://docs.djangoproject.com/en/4.0/topics/db/multi-db/
https://books.agiliq.com/projects/django-multi-tenant/en/latest/isolated-database.html
