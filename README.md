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

Create a file called `compose.yaml` in your project directory and paste the following:

```yml
services:
  web:
    build: .
    ports:
      - "8000:5000"
  redis:
    image: "redis:alpine"
```

#### The Compose file

[The Compose file](https://docs.docker.com/compose/compose-file/03-compose-file/#compose-file)

The default path for a Compose file is `compose.yaml` or `compose.yml`; (`.yaml`) is preffered, the file
it's placed in the working direcoty. Compose also supports `docker-compose.yaml`or `docker-compose.yml` but they are use for backwards compatibility of earlier versions.

#### Using Compose:

[Docker Compose Features uses](https://docs.docker.com/compose/features-uses/)

Using Compose is essentially a three-step process:

1. Define your app's environment with a `Dockerfile` so it can be reproduced anywhere.
2. Define the `services` that make up your app in a c`compose.yaml` file so they can be `run together in an isolated environment`
3. Run `docker compose up` and the Docker compose command starts and runs your entire app.

[The Compose application model](https://docs.docker.com/compose/compose-file/02-model/#the-compose-application-model)

Compose lets you define an application with containers for multiple platforms, with adequate shared resources and communication channels.

[Docker Compose Service](https://docs.docker.com/compose/compose-file/05-services/)

- **Services:** Are computing components of an application, is an abstract definition of a computing resource within an application which can be scaled or replaced independently from other components.

[Docker Compose version, and names for services](https://docs.docker.com/compose/compose-file/04-version-and-name/)

- **Version:** `When I started learning docker a few years ago I used to use the `version` ` property, I was thought to use this but without further explanation it was like a default thing you have to use when writting docker compose.

Now documentation states that `version` property it's only informative, you can define it but it will only be used for backward compatibilty.

Compose doesn't use `version` to select an exact schema to validate the Compose file, but prefers the
most recent schema when it's implemented.


This docker compose file defines two services: `web` and `redis`.

- The `web` service uses an image that's built from the `Dockerfile` in the current directory.
- It then binds the container and the host machine to the exposed port, 8000
- This example service uses the default port for the Flask web server, 5000.

[Compose Build subsection](https://docs.docker.com/compose/compose-file/build/#compose-build-specification)

The Compose Build Specification lets you define the build process within a Compose file in a portable way.

- The `redis` service uses a public `Redis` image pulled from the Docker Hub registry.

[Compose service image attribute](https://docs.docker.com/compose/compose-file/05-services/#image)

- `image` specifies the image to start the container from. Pulling the image is the default behavior.

[Docker compose ports attribute](https://docs.docker.com/compose/compose-file/05-services/#ports)

- `ports:` Exposes container ports. [HOST:]CONTAINER[/PROTOCOL]
           If no host port specified, Compose automatically allocates any unassigned port of the host.
- `HOST:CONTAINER` should always be specified as a (quoted) string, to avoid conflicts with yaml.

## Step 4: Build and run your app with Compose.

1. From your project directory, start up your application by running `docker compose up`.

```docker
$ docker compose up

[+] Running 9/9
 ✔ redis 8 layers [⣿⣿⣿⣿⣿⣿⣿⣿]      0B/0B      Pulled                                                               12.4s
   ✔ 661ff4d9561e Pull complete                                                                                    4.3s
   ✔ 0235bd0a3e9b Pull complete                                                                                    4.6s
   ✔ 003d22e43376 Pull complete                                                                                    5.9s
   ✔ 4f3278ab7ee0 Pull complete                                                                                    6.0s
[+] Building 48.8s (10/12)

.
.
.

 ✔ Network composetest_default    Created                                                                          0.2s
 ✔ Container composetest-redis-1  Created                                                                          0.6s
 ✔ Container composetest-web-1    Created                                                                          0.6s
Attaching to composetest-redis-1, composetest-web-1
.
.
.
 ✔ Network composetest_default    Created                                                                          0.2s
 ✔ Container composetest-redis-1  Created                                                                          0.6s
 ✔ Container composetest-web-1    Created                                                                          0.6s
Attaching to composetest-redis-1, composetest-web-1
```

Compose pull a `Redis` image, builds an image for your code, and starts the services you defined.
In this case, `the code is statically copied into the image at build time`.

2. Enter `http://localhost:8000/` in a browser to see the application running.

![image](https://github.com/lestersantos/try-docker-compose-v2/assets/30423581/1bb7c9ce-66c4-4f0b-8d0a-574acf68b44d)

3. Refresh the page. The number should increment.

![image](https://github.com/lestersantos/try-docker-compose-v2/assets/30423581/3276bfe2-04fa-4e2e-8e8c-0fa7bb96d88d)

4. Open another terminal window, and type `docker images` to list local images.

```bash
CONTAINER ID   IMAGE             COMMAND                  CREATED       STATUS                    PORTS                    NAMES
e2aaafb967da   redis:alpine      "docker-entrypoint.s…"   3 hours ago   Up 7 seconds              6379/tcp                 composetest-redis-1
8221eb7d005f   composetest-web   "flask run"              3 hours ago   Up 7 seconds              0.0.0.0:8000->5000/tcp   composetest-web-1
9ccb93d200ca   python            "/bin/bash"              4 days ago    Exited (130) 4 days ago                            laughing_nobel
```

5. Stop the application, either by running `docker compose down` from within your project directory in the
second terminal, or by hitting `CTRL+C` in the orginal terminal where you started the app.

```bash
composetest-web-1    | Press CTRL+C to quit
composetest-web-1    | 172.18.0.1 - - [10/Jan/2024 20:13:24] "GET / HTTP/1.1" 200 -
composetest-web-1    | 172.18.0.1 - - [10/Jan/2024 20:28:59] "GET / HTTP/1.1" 200 -
[+] Stopping 2/2ing... (press Ctrl+C again to force)
 ✔ Container composetest-redis-1  Stopped                                                                          0.5s
 ✔ Container composetest-web-1    Stopped                                                                         10.4s
canceled
```
## Step 5: Edit the Compose file to add a bind mount.

Edit the `compose.yaml` file in your project directory to add a bind mount for the `web` service:

```yaml
services:
  web:
    build: .
    ports:
      - "8000:5000"
    volumes:
      - .:/code
    environment:
      FLASK_DEBUG: "true"
  redis:
    image: "redis:alpine"
```

1. The new `volumes` key mounts the project directory (current directory) or the host to `/code` inside
the container, allowing you to modify the code on the fly, without have to rebuild the image.
2. The environment key sets the `FLASK_DEBUG` environment variable, which tells `flask run` to run in
development mode and reload the code on change.

[Docker volumes vs Bind mounts](https://docs.docker.com/storage/volumes/)

`Volumes` are the preffered mechanism for persisting data generated by and used by Docker
containers.

`Bind mounts` and `Volumes` attributes work different, in my words, `Bind mounts` are dependent of the 
OS, you attach a directory to a container and the path is specific for wach type of OS, while `volumes` are completly managed by Docker.

[Docker Bind mounts](https://docs.docker.com/storage/bind-mounts/)

When you use a `bind mount`, a `file or directory` on the host machine `is mounted` into a container.
with volumes, the file or directory does not need to exist on the Docker host already, it's created on demand.

[Docker compose volumes](https://docs.docker.com/compose/compose-file/05-services/#volumes)

`Volumes` define mount host paths or named volues that are accessible by service containers.
You can use `volumes` to define multiple types of mounts: `volume`, `binds`, `tmpfs`, or `npipe`.

- If the mount is only used by a `single service`, it can be declared as part of the
`service definition`.
- If you'd like to reuse a volume across multiple services, a named `volume top-level` element most
be declared.
- Syntax `VOLUME:CONTAINER_PATH`.

## Step 6: Re-build and run the app with Compose

Now that changes where made to `compose.yaml` file, from your project directory, type `docker compose up` to build the app with the updated Compose file, and run it. We didn't touch the code just the definition of our services in the docker compose file.

```bash
$ docker compose up
[+] Running 2/2
 ✔ Container composetest-redis-1  Created                                                                                                                                            0.0s 
 ✔ Container composetest-web-1    Recreated                                                                                                                                          0.3s 
Attaching to redis-1, web-1
.
.
.
web-1    |  * Running on all addresses (0.0.0.0)
web-1    |  * Running on http://127.0.0.1:5000
web-1    |  * Running on http://172.18.0.3:5000
web-1    | Press CTRL+C to quit
web-1    |  * Restarting with stat
web-1    |  * Debugger is active!
web-1    |  * Debugger PIN: 124-770-505
```

## Step 7: Update the application

The application code is now mounted into the container using a volume, you can make changes to its code
and see the changes instantly, without having to rebuild the image.

When we first created our image, the initial code was copied from the project directory to the workdir `/code` inside the container, now the image is alreay created, but thanks to the `bind mount` we can
make changes to the code in our project directory and they will be reflected instantly in our running
container.

- Change the greeting in `app.py` and save it. For example, change the `Hello World!` message to
`Hello from Docker!`.

```python
return 'Hello from Docker! I have been seen {} times.\n'.format(count)
```

- Refresh the app in your browser. The greeting should be updated, and the counter should still be
incrementing.

## The END

This is the end of the tutorial for **Docker Compose V2**. This tutorial continues with 

[Step 8: Experiment with some other commands](https://docs.docker.com/compose/gettingstarted/#step-8-experiment-with-some-other-commands)

Where we can experiment with other `docker compose [OPTIONS] COMMAND`. 

The reference for the docker compose commands can be found here 

[Overview of docker compose CLI](https://docs.docker.com/compose/reference/)