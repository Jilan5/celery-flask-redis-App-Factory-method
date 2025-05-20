# Flask App Factory with Celery and Redis

## Table of Contents
- [Introduction](#introduction)
- [Part 1: Setting Up the Flask App Factory](#part-1-setting-up-the-flask-app-factory)
  - [1.1 Setting Up the Environment](#11-setting-up-the-environment)
  - [1.2 Creating the App Factory](#12-creating-the-app-factory)
  - [1.3 Configuration](#13-configuration)
  - [1.4 Adding a Blueprint and Model](#14-adding-a-blueprint-and-model)
  - [1.5 Register the Blueprint](#15-register-the-blueprint)
  - [1.6 Database Operations](#16-database-operations)
- [Part 2: Adding Celery Integration](#part-2-adding-celery-integration)
  - [2.1 Update Configuration for Celery](#21-update-configuration-for-celery)
  - [2.2 Create Celery Utilities](#22-create-celery-utilities)
  - [2.3 Update App Factory for Celery](#23-update-app-factory-for-celery)
  - [2.4 Create a Celery Task](#24-create-a-celery-task)
  - [2.5 Update App Entry Point](#25-update-app-entry-point)
  - [2.6 Test Celery](#26-test-celery)
- [Part 3: Containerizing with Docker Compose](#part-3-containerizing-with-docker-compose)
  - [3.1 Create Docker Compose Configuration](#31-create-docker-compose-configuration)
  - [3.2 Create Environment Variables](#32-create-environment-variables)
  - [3.3 Create Docker Configuration Files](#33-create-docker-configuration-files)
  - [3.4 Create Entry and Start Scripts](#34-create-entry-and-start-scripts)
  - [3.5 Add a Simple Route](#35-add-a-simple-route)
  - [3.6 Build and Run with Docker Compose](#36-build-and-run-with-docker-compose)
  - [3.7 Testing with Docker Compose](#37-testing-with-docker-compose)
  - [3.8 Useful Docker Commands](#38-useful-docker-commands)
- [Final Project Structure](#final-project-structure)
- [Conclusion](#conclusion)

## Introduction

The App Factory pattern is a design pattern for Flask applications that allows for better modularization, testing, and maintainability. This tutorial combines this pattern with Celery for handling background tasks and Redis as a message broker.

## Part 1: Setting Up the Flask App Factory

### 1.1 Setting Up the Environment

First, let's set up our development environment:

```bash
# Update system and install Python virtual environment
sudo apt update
apt install python3.8-venv

# Set up Redis using Docker
docker run -p 6379:6379 --name some-redis -d redis

# Test Redis connection
docker exec -it some-redis redis-cli ping
# Should return: PONG

# Create project directory and virtual environment
mkdir flask-celery-project && cd flask-celery-project
python3 -m venv env
source env/bin/activate

# Create and install requirements
cat > requirements.txt << EOF
Flask==3.0.3
celery==5.4.0
redis==5.0.8
flower==2.0.1
Flask-Migrate==4.0.7
Flask-SQLAlchemy==3.1.1
Flask-CeleryExt==0.5.0
psycopg==3.2.1
EOF

pip install -r requirements.txt
```

### 1.2 Creating the App Factory

Create a project directory structure:

```bash
flask-celery-project/
├── app.py
└── project/
    ├── __init__.py
    └── config.py
```

First, let's create the factory function in "project/__init__.py":

```python name=project/__init__.py
import os

from flask import Flask
from flask_migrate import Migrate
from flask_sqlalchemy import SQLAlchemy


# instantiate the extensions
db = SQLAlchemy()
migrate = Migrate()


def create_app(config_name=None):
    if config_name is None:
        config_name = os.environ.get("FLASK_CONFIG", "development")

    # instantiate the app
    app = Flask(__name__)

    # set config
    from project.config import config
    app.config.from_object(config[config_name])

    # set up extensions
    db.init_app(app)
    migrate.init_app(app, db)

    # shell context for flask cli
    @app.shell_context_processor
    def ctx():
        return {"app": app, "db": db}

    return app
```

Now create the app entry point at "app.py":

```python name=app.py
from project import create_app

app = create_app()
```

### 1.3 Configuration

Create a configuration file "project/config.py":

```python name=project/config.py
import os
from pathlib import Path

class BaseConfig:
    """Base configuration"""
    BASE_DIR = Path(__file__).parent.parent

    TESTING = False
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SQLALCHEMY_DATABASE_URI = os.environ.get("DATABASE_URL", f"sqlite:///{BASE_DIR}/db.sqlite3")


class DevelopmentConfig(BaseConfig):
    """Development configuration"""
    DEBUG = True


class ProductionConfig(BaseConfig):
    """Production configuration"""
    DEBUG = False


config = {
    "development": DevelopmentConfig,
    "production": ProductionConfig,
}
```

### 1.4 Adding a Blueprint and Model

Create a users module with a blueprint and model:

```bash
project/
└── users/
    ├── __init__.py
    └── models.py
```
In name=project/users/__init__.py file:
```python name=project/users/__init__.py
from flask import Blueprint

users_blueprint = Blueprint("users", __name__, url_prefix="/users", template_folder="templates")

from . import models  # noqa
```
In name=project/users/models.py file:
```python name=project/users/models.py
from project import db


class User(db.Model):
    __tablename__ = "users"

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    username = db.Column(db.String(128), unique=True, nullable=False)
    email = db.Column(db.String(128), unique=True, nullable=False)

    def __init__(self, username, email, *args, **kwargs):
        self.username = username
        self.email = email
```

### 1.5 Register the Blueprint

Update the factory function in "project/__init__.py" to register the blueprint:

```python
def create_app(config_name=None):
    # ... existing code ...
    
    # register blueprints
    from project.users import users_blueprint
    app.register_blueprint(users_blueprint)
    
    # ... existing code ...
```

### 1.6 Database Operations

Initialize the database and create tables:

```bash
FLASK_APP=app.py flask db init
FLASK_APP=app.py flask db migrate -m "Initial migration."
FLASK_APP=app.py flask db upgrade
```

Test the database with the Flask shell:

```bash
FLASK_APP=app.py flask shell
```

```python
>>> from project.users.models import User
>>> user = User(username='test1', email='test1@example.com')
>>> db.session.add(user)
>>> db.session.commit()
>>> User.query.all()
[<User 1>]
>>> User.query.first().username
'test1'
```

## Part 2: Adding Celery Integration

### 2.1 Update Configuration for Celery

Add Celery configuration to "project/config.py":

```python
class BaseConfig:
    # ... existing config ...
    
    CELERY_BROKER_URL = os.environ.get("CELERY_BROKER_URL", "redis://127.0.0.1:6379/0")
    CELERY_RESULT_BACKEND = os.environ.get("CELERY_RESULT_BACKEND", "redis://127.0.0.1:6379/0")
```

### 2.2 Create Celery Utilities

Create a helper file "project/celery_utils.py":

```python name=project/celery_utils.py
from celery import current_app as current_celery_app


def make_celery(app):
    celery = current_celery_app
    celery.config_from_object(app.config, namespace="CELERY")

    return celery
```

### 2.3 Update App Factory for Celery

Update "project/__init__.py" to integrate Celery:

```python name=project/__init__.py
import os

from flask import Flask
from flask_celeryext import FlaskCeleryExt  # new
from flask_migrate import Migrate
from flask_sqlalchemy import SQLAlchemy

from project.celery_utils import make_celery  # new
from project.config import config


# instantiate the extensions
db = SQLAlchemy()
migrate = Migrate()
ext_celery = FlaskCeleryExt(create_celery_app=make_celery)  # new


def create_app(config_name=None):
    if config_name is None:
        config_name = os.environ.get("FLASK_CONFIG", "development")
    
    app = Flask(__name__)
    app.config.from_object(config[config_name])

    db.init_app(app)
    migrate.init_app(app, db)
    ext_celery.init_app(app)  # new

    from project.users import users_blueprint
    app.register_blueprint(users_blueprint)

    @app.shell_context_processor
    def ctx():
        return {"app": app, "db": db}

    return app
```

### 2.4 Create a Celery Task

Create a task file "project/users/tasks.py":

```python name=project/users/tasks.py
from celery import shared_task


@shared_task
def divide(x, y):
    import time
    time.sleep(5)
    return x / y
```

Update "project/users/__init__.py" to import the tasks:

```python
from flask import Blueprint

users_blueprint = Blueprint("users", __name__, url_prefix="/users", template_folder="templates")

from . import models, tasks  # noqa
```

### 2.5 Update App Entry Point

Update "app.py" to export the Celery instance:

```python name=app.py
from project import create_app, ext_celery

app = create_app()
celery = ext_celery.celery
```

### 2.6 Test Celery

Run a Celery worker:

```bash
celery -A app.celery worker --loglevel=info
```

Then in another terminal, test with Flask shell:

```bash
FLASK_APP=app.py flask shell
```

```python
>>> from project.users.tasks import divide
>>> divide.delay(1, 2)  # Executes task asynchronously
```

## Part 3: Containerizing with Docker Compose

### 3.1 Create Docker Compose Configuration

Create a "docker-compose.yml" file in the project root:

```yaml name=docker-compose.yml
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: ./compose/local/flask/Dockerfile
    image: flask_celery_example_web
    command: /start
    volumes:
      - .:/app
    ports:
      - 5010:5000
    env_file:
      - .env/.dev-sample
    environment:
      - FLASK_APP=app
    depends_on:
      - redis
      - db

  db:
    image: postgres:16-alpine
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    environment:
      - POSTGRES_DB=flask_celery
      - POSTGRES_USER=flask_celery
      - POSTGRES_PASSWORD=flask_celery

  redis:
    image: redis:7-alpine

  celery_worker:
    build:
      context: .
      dockerfile: ./compose/local/flask/Dockerfile
    image: flask_celery_example_celery_worker
    command: /start-celeryworker
    volumes:
      - .:/app
    env_file:
      - .env/.dev-sample
    environment:
      - FLASK_APP=app
    depends_on:
      - redis
      - db

  celery_beat:
    build:
      context: .
      dockerfile: ./compose/local/flask/Dockerfile
    image: flask_celery_example_celery_beat
    command: /start-celerybeat
    volumes:
      - .:/app
    env_file:
      - .env/.dev-sample
    environment:
      - FLASK_APP=app
    depends_on:
      - redis
      - db

  flower:
    build:
      context: .
      dockerfile: ./compose/local/flask/Dockerfile
    image: flask_celery_example_celery_flower
    command: /start-flower
    volumes:
      - .:/app
    env_file:
      - .env/.dev-sample
    environment:
      - FLASK_APP=app
    ports:
      - 5557:5555
    depends_on:
      - redis
      - db

volumes:
  postgres_data:
```

### 3.2 Create Environment Variables

Create an environment file at ".env/.dev-sample":

```bash name=.env/.dev-sample
FLASK_DEBUG=1
FLASK_CONFIG=development
DATABASE_URL=postgresql+psycopg://flask_celery:flask_celery@db/flask_celery
SECRET_KEY=my_precious
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/0
```

### 3.3 Create Docker Configuration Files

Create the Dockerfile at "compose/local/flask/Dockerfile":

```dockerfile name=compose/local/flask/Dockerfile
FROM python:3.11-slim-buster

ENV PYTHONUNBUFFERED 1
ENV PYTHONDONTWRITEBYTECODE 1

RUN apt-get update \
  && apt-get install -y build-essential \
  && apt-get install -y libpq-dev \
  && apt-get install -y telnet netcat \
  && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
  && rm -rf /var/lib/apt/lists/*

COPY ./requirements.txt /requirements.txt
RUN pip install -r /requirements.txt

COPY ./compose/local/flask/entrypoint /entrypoint
RUN sed -i 's/\r$//g' /entrypoint
RUN chmod +x /entrypoint

COPY ./compose/local/flask/start /start
RUN sed -i 's/\r$//g' /start
RUN chmod +x /start

COPY ./compose/local/flask/celery/worker/start /start-celeryworker
RUN sed -i 's/\r$//g' /start-celeryworker
RUN chmod +x /start-celeryworker

COPY ./compose/local/flask/celery/beat/start /start-celerybeat
RUN sed -i 's/\r$//g' /start-celerybeat
RUN chmod +x /start-celerybeat

COPY ./compose/local/flask/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /start-flower
RUN chmod +x /start-flower

WORKDIR /app

ENTRYPOINT ["/entrypoint"]
```

### 3.4 Create Entry and Start Scripts

Create the following scripts:

```bash name=compose/local/flask/entrypoint
#!/bin/bash

set -o errexit
set -o pipefail
set -o nounset

postgres_ready() {
python << END
import sys

import psycopg
import urllib.parse as urlparse
import os

url = urlparse.urlparse(os.environ['DATABASE_URL'])
dbname = url.path[1:]
user = url.username
password = url.password
host = url.hostname
port = url.port

try:
    psycopg.connect(
        dbname=dbname,
        user=user,
        password=password,
        host=host,
        port=port
    )
except psycopg.OperationalError:
    sys.exit(-1)
sys.exit(0)

END
}
until postgres_ready; do
  >&2 echo 'Waiting for PostgreSQL to become available...'
  sleep 1
done
>&2 echo 'PostgreSQL is available'

exec "$@"
```

```bash name=compose/local/flask/start
#!/bin/bash

set -o errexit
set -o pipefail
set -o nounset

flask db upgrade
flask run --host=0.0.0.0
```

```bash name=compose/local/flask/celery/worker/start
#!/bin/bash

set -o errexit
set -o nounset

celery -A app.celery worker --loglevel=info
```

```bash name=compose/local/flask/celery/beat/start
#!/bin/bash

set -o errexit
set -o nounset

rm -f './celerybeat.pid'
celery -A app.celery beat -l info
```

```bash name=compose/local/flask/celery/flower/start
#!/bin/bash

set -o errexit
set -o nounset

worker_ready() {
    celery -A app.celery inspect ping
}

until worker_ready; do
  >&2 echo 'Celery workers not available'
  sleep 1
done
>&2 echo 'Celery workers is available'

celery -A app.celery  \
    --broker="${CELERY_BROKER_URL}" \
    flower
```

### 3.5 Add a Simple Route

Update "app.py" to include a simple route:

```python
from project import create_app, ext_celery

app = create_app()
celery = ext_celery.celery

@app.route("/")
def hello_world():
    return "Hello, World!"
```

### 3.6 Build and Run with Docker Compose

Build the Docker images:

```bash
docker compose build
```

Start the containers:

```bash
docker compose up -d
```

### 3.7 Testing with Docker Compose

• Test the Flask app: Visit http://localhost:5010/ to see the "Hello, World!" message

• Test Celery tasks:

```bash
# Enter the Flask shell in the web container
docker compose exec web flask shell
```

```python
>>> from project.users.tasks import divide
>>> divide.delay(1, 2)
```

• Check the Flower dashboard: Visit http://localhost:5557/ to monitor Celery tasks

### 3.8 Useful Docker Commands

```bash
# View logs
docker compose logs -f

# Access shell in a running container
docker compose exec web bash

# Run a command in a new container
docker compose run --rm web bash
```

## Final Project Structure

```bash
flask-celery-project/
├── app.py
├── celerybeat-schedule
├── compose/
│   └── local/
│       └── flask/
│           ├── Dockerfile
│           ├── celery/
│           │   ├── beat/
│           │   │   └── start
│           │   ├── flower/
│           │   │   └── start
│           │   └── worker/
│           │       └── start
│           ├── entrypoint
│           └── start
├── docker-compose.yml
├── .env/
│   └── .dev-sample
├── migrations/
├── project/
│   ├── __init__.py
│   ├── celery_utils.py
│   ├── config.py
│   └── users/
│       ├── __init__.py
│       ├── models.py
│       └── tasks.py
└── requirements.txt
```

## Conclusion

You've now created a Flask application using the App Factory pattern with Celery integration for background task processing and Redis as the message broker. This setup provides a solid foundation for building scalable web applications with asynchronous capabilities.
