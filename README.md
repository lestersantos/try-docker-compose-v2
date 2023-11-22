# Getting started with Docker Compose

## Enviroment Setup

By the time of the tutorial we are using the following setup:

- `Docker Compose Version v2.19.1`
- `Docker Desktop 4.21.1`
- `Docker Engine V24.02`
- `WSL 2 Backend` Version 1.2.5.0
    - Kernel version: 5.15.90.1
    - Windows version: 10.0.19045.3570

## Step 1: Define the application dependencies

### 1. Create a directory for the project
```bash
mkdir composetest
cd composetest
```

### 2. Create a file called `app.py` in your project directory and paste the following code in:
```python
import time

import redis
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    retries = 5
    while True:
        try:
            return cache.incr('hits')
        except redis.exceptions.ConnectionError as exc:
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)
```

### A Minimal Flask Application

[Flask Quickstart](https://flask.palletsprojects.com/en/2.1.x/quickstart/)

1. First we imported the **Flask** class. 
```python
from flask import Flask
```
2. Next we create an instance of the of this class. The first argument is the name of
the application's module or package. `__name__` is a convenient shortcut for this that
is appropiate for most cases. This is needed so that Flask knows where to look for resources such as
templates and static files.

```python
app = Flask(__name__)
```

3. We then use the `route()` decorator to tell Flask what URL should tirgger our function.
```python
@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times.\n'.format(count)
```
4. The function returns the message we want to display in the user's browser. The default
content type is HTML, so HTML in the string will be rendered by the browser.

### Redis

[Introduction to Redis](https://redis.io/docs/about/)

Redis is an open source (BSD licensed), in-memory **data structure store** used as a database, cache, message broker, and streaming engine.

#### Connect to Redis

You can connect to Redis in the following ways:
 - With th redis-cli command line tool
 - Use RedisInsight as a graphical user interface 
 - Via a **client library** for your programming language

#### Python Client Library

[Connect to a redis database with client libraries ](https://redis.io/docs/connect/)

It's easy to connect your application to a Redis database using
the official client libraries.

In this app we are using the **Python library**.

[Python guide](https://redis.io/docs/connect/clients/python/)

Install Redis and the Redis client, then connect your Python application
to a Redis database.

##### Connecting to a default Redis instance, running locally.

[Connecting to a default Redis instance, specifying a host and port](https://redis.readthedocs.io/en/stable/examples/connection_examples.html#Connecting-to-a-redis-instance,-specifying-a-host-and-port-with-credentials.)

```py
import redis
cache = redis.Redis(host='redis', port=6379)
```
- In this example, `'redis'` is the hostname of the redis container on the application's network. We use the default port for Redis, `6379`.

### 3. Create another file called `requirements.txt` in your project directory and paste the following code in:

```txt
flask
redis
```

## Step 2: Create a Dockerfile

The Dockerfile is used to build a Docker image. The image contains all the dependencies the Python application requires, including Python itself.

In your project directory, create a file named `Dockerfile` and paste the following code in:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.7-alpine
WORKDIR /code
ENV FLASK_APP=app.py
ENV FLASK_RUN_HOST=0.0.0.0
RUN apk add --no-cache gcc musl-dev linux-headers
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
EXPOSE 5000
COPY . .
CMD ["flask", "run"]
```
- Line 1: We write a comment in a docker file with hash(also called `pound`) sign `#`.

    `# syntax=docker/dockerfile:1`

- Line 2: Build an image starting with the Python 3.7 image.

    `FROM python:3.7-alpine`

- Line 3: Set the working directory to `/code`.

    `WORKDIR /code`

- Line 4,5: Set the environment variables used by the `flask` command
    - `ENV FLASK_APP=app.py`
    - `ENV FLASK_RUN_HOST=0.0.0.0`

- Line 6: Install gcc and other dependencies.
    
    `RUN apk add --no-cache gcc musl-dev linux-headers`

- Line 7: Copy `requirements.txt` and install the Python dependencies.
    
    `COPY requirements.txt requirements.txt`

- Line 8: Install the Python dependencies.

    `RUN pip install -r requirements.txt`

- Line 9: Add metadata to the image to describe that the container is listening on port 5000.

    `EXPOSE 5000`

- Line 10: Copy the current directory `.`  in the project to the workdir `.` in the image. `COPY <src> <dest>`

    `COPY . .`

- Line 11: Set the default command for the container to `flask run`.

[The CMD command](https://docs.docker.com/engine/reference/builder/#cmd)

The `CMD` has three forms, we are using the exec form:

`CMD ["executable","param1","param2"]`

    `CMD ["flask","run"]`

The main purpose of a `CMD` is to provide defaults for an executing container. If you ignore the executable, in this case `flask`, you must specify an `ENTRYPOINT` instruction as well.

-**NOTE:** Do not confuse `RUN` with `CMD`. `RUN` actually runs a command and commits the result;
`CMD` does not execute anything at build time, but specifies the intended command for the image.

## Step 3: Define services in a Compose file.

Create a file called `compose.yaml`