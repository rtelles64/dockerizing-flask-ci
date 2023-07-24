# Robust Continuous Integration with Docker

> **NOTE**
>
> This project follows Real Python's [Build Robust Continuous Integration with Docker and Friends][dockerizing-flask-ci] tutorial.

Docker containers help facilitate the _continuous integration_ (CI) process by providing a consistent environment where you can test and ship code on each commit.

This tutorial looks at how to use Docker to create a robust CI pipeline for a Flask web app. The steps followed in this tutorial cover developing and testing an application locally, containerizing it, orchestrating containers using _Docker Compose_, and defining a CI pipeline using _GitHub Actions_.

The resources used in this tutorial are:

- Redis server
- Flask web app
- Docker/Docker Compose
- Github Actions

> **NOTE**
>
> It's recommended to have some experience with the following concepts:
>
> - [Python web development][web-development]
> - [test automation][test-automation]
> - [Redis with Python][python-redis]
> - [version control with Github][github]

You can download a sample web application [here][flask-app-resources].

## Architecture Overview

By the end of this tutorial, you'll have designed a Flask web application for tracking page views stored persistently in a Redis data store. This will be a multi-container application orchestrated using Docker Compose. You'll be able to build and test locally, as well as in the cloud, opening up the pathway to continuous integration.

The architecture for this project is detailed below:

![Architecture Overview](./images/docker-flask-ci-architecture.png)

This application consists of two Docker containers:

- The first container will run a Flask app which responds to HTTP requests and updating the number of page views

- The second container will run a Redis instance for storing the page view data persistently in a local volume on the host machine

The great thing is Docker is all that's required to run this application! We'll set this up in the next section.

## Set Up Docker

Docker enables the ability to run applications anywhere in consistent and reproducible environments with little or no configuration. It can package application code and dependencies into a single artifact called a _container_. We'll use Docker to simulate a virtual production environment on your local machine during development and on a continuous integration server.

There are 2 methods to install Docker:

1. [Docker Engine][docker-engine]
2. [Docker Desktop][docker-desktop]

**Docker Engine** provides an extra level of control but requires comfortability with the command line. On the other hand, **Docker Desktop** provides an intuitive GUI for managing containers and images. Docker Desktop still comes with a command line interface (CLI) for more advanced operations.

> **NOTE**
>
> Out of the box, the desktop application comes with Docker Compose, which we'll use for orchestrating containers for continuous integration.

While it's possible to have both Docker Engine and Docker Desktop installed, you should generally avoid using them together to minimize the risk of any potential interference between their virtual networks or port bindings.

To verify you have Docker, or have successfully installed it on your system, open a terminal window and run the following command:

```shell
docker --version
Docker version 24.0.2, build cb74dfc
```

There should be a version along with the build number. If you don't see this, then you'll need to install Docker.

In general, before using Docker to assist in continuous integration, you'll need to create a rudimentary application, web or otherwise.

## Develop a Flask App &mdash; Page View Tracker

The Flask app in this tutorial will keep track of the total number of page views and display that number to the user with each request:

![Page View Tracker](./images/screenshot-browser.png)

The current state of the application will be saved in a Redis data store, which is commonly used for caching and other types of data persistence. In this way, stopping the web server won't reset the view count. Redis can be considered a type of database.

> See the [resources link][flask-app-resources] if you're not interested in building this application from scratch. Either way, it'll be useful to cross reference in case you get stuck.

### Prepare the Environment

As with every Python project, it's best practice to create a virtual environment to isolate the dependencies. This will ensure that the dependencies for this project don't interfere with other projects on your system and help to maintain the smallest possible container image for your app.

In the terminal run the following commands:

```shell
mkdir page-tracker; cd page-tracker

python3.11 -m venv tracker-app-env
```

- The first line creates a new directory for our app and changes the working directory to it.

- The second line creates a virtual environment called `tracker-app-env` using Python 3.11 and the `venv` module (indicated by the `-m` flag). You can replace `python3.11` with `python3` (which uses the default Python version installed on your system) or whatever version of Python you prefer to use for this environment.

To activate the virtual environment, run the following command:

```shell
source tracker-app-env/bin/activate

(tracker-app-env) $ pip install --upgrade pip
```

- After activating the virtual environment, you should see the name of the environment in parentheses before the prompt.

- You should upgrade to the latest version of `pip` to avoid any potential issues with dependency resolution when installing Python packages.

In this tutorial, you'll use the modern way of specifying dependencies and metadata through a [`pyproject.toml`][pyproject-toml] configuration file and [setuptools][setuptools] as the [build backend][build-backend]. Additionally, you'll follow the [`src` layout][src-layout] by placing your app's source code in a separate `src/` subdirectory to better organize the files in your project. This makes it straightforward to package your code without the automated tests we'll add later.

To begin, create the following file tree:

```shell
page-tracker
├── pyproject.toml
├── requirements.txt
├── tracker-app-env
└── src
    └── page_tracker
        ├── __init__.py
        └── app.py
```

Based on the project structure, we'll only have one Python module, `app`, defined in a package called `page_tracker`, sitting inside the `src` directory. The `requirements.txt` file will specify dependencies for the project in order to achieve [repeatable installs][repeatable-installs].

> **NOTE**
>
> The original tutorial uses a `constraints.txt` file to manage package dependencies and versions. In this project, we use `requirements.txt` as it is simpler and we utilize the `pip-chill` package to only list the minimum required packages for app funcitonality.

Since this project will depend on Flask and Redis, we can declare that in the `pyproject.toml` file:

```toml
# pyproject.toml

[build-system]
requires = ["setuptools>=67.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "page-tracker"
version = "1.0.0"
dependencies = [
    "Flask",
    "redis",
]
```

You typically don't specify dependency versions here. Instead, you would list them in a requirements or a constraints file. The first one tells `pip` what packages to install, the latter enforces specific versions of transitive dependencies.

Before coding the web app, we need to prepare a local Redis server to connect to over a network.

### Run a Redis Server Through Docker

> **RECOMMENDED**
>
> If you are unfamiliar with how to use Redis with Python, it is recommended to familiarize yourself with useful commands by going through the [How to Use Redis with Python][python-redis] tutorial.

_Redis_ is a portmanteau of the words _remote dictionary server_, which accurately conveys its purpose as a remote, in-memory data structure store. Being a key-value store, Redis is a remote Python dictionary that you can connect to from anywhere. It's also considered one of the most popular NoSQL databases used in many different contexts. Frequently, it serves the purpose of a cache on top of a relational database.

> **NOTE**
>
> While Redis keeps all of its data in [_volatile memory_][volatile-memory], which makes it extremely fast, the server comes with a variety of [persistence options][redis-persistence]. They can ensure different levels of data durability in case of a power outage or reboot. However, configuring Redis correctly often proves difficult, which is why many teams decide to use a managed service outsourced to cloud providers.

Installing Redis on your local machine is simple, but running it through Docker is even simpler and more elegant, assuming you've installed and configured Docker before. When you run a service, such as Redis, in a Docker container, it remains isolated from the rest of your system without causing clutter or hogging system resources like network port numbers, which are limited.

To run Redis without installing it on your host machine you can run a new Docker container from the [official Redis image][redis-image] by invoking the following command:

```shell
docker run -d --name redis-server redis
Unable to find image 'redis:latest' locally
latest: Pulling from library/redis
3ae0c06b4d3a: Already exists 
...
Status: Downloaded newer image for redis:latest
```

- This creates a new Docker container based on the latest version of the `redis` image, with the custom name `redis-server`, which you'll refer to later. The container is running in the background in detached mode (`-d`). When you run this command for the first time, Docker will pull the corresponding Docker image from Docker Hub, which is the official repository for Docker images, similar to PyPI.

Assuming no errors, the Redis server (as a container) should be up and running and active in the background (in detached mode, `-d`). To verify this, you can list your Docker containers using the `docker container ls` command or the alias `docker ps`:

```shell
docker ps
CONTAINER ID   IMAGE   ...   STATUS          PORTS      NAMES
f0f84362b087   redis   ...   Up 13 minutes   6379/tcp   redis-server
```

- Here, you can see useful values like the running container ID, the image it was based off of, its status, the alias we gave it (`redis-server`), and the TCP port number `6379`, which is the default used by Redis.

Next, we'll try connecting to the Redis server in various ways.

### Test the Connection to Redis

On the [overview page][redis-image] of the official Redis image, you'll find instructions on how to connect to a Redis server running in a Docker container. Specifically, this page talks about using the dedicated interactive [Redis CLI][redis-cli] that comes with the Docker image.

You can start another Docker container from the same `redis` image, but this time, set the container's entry point to the `redis-cli` command instead of the default Redis server binary. When you setup multiple containers to work together, you should use [Docker networks][docker-networks], which require a few extra steps to configure.

First, create a new user-defined [**bridge network**][docker-bridge-network] named after your project, for example:

```shell
docker network create page-tracker-network
005d5953e05f03e01a8bb5c66c35265ef091eae7d363387fa9a643049cabddf0
```

By defining a virtual network, you can hook up as many Docker containers as you like and let them discover each other through descriptive names. You can list the networks that you've created with the following command:

```shell
docker network ls
NETWORK ID     NAME                   DRIVER    SCOPE
dda284158bf5   bridge                 bridge    local
3f1eb84a8437   host                   host      local
135abcc95615   none                   null      local
005d5953e05f   page-tracker-network   bridge    local
```

Next, connect your existing `redis-server` container to this new virtual network, and specify the same network for the Redis CLI when you start its corresponding container:

```shell
docker network connect page-tracker-network redis-server
docker run --rm -it \
           --name redis-client \
           --network page-tracker-network \
           redis redis-cli -h redis-server
```

- The `--rm` flag tells Docker to remove the created container as soon as you terminate it since this is a temporary (or [_ephemeral_][ephemeral-container]) container that you don't need to start ever again.
- The `-i` and `-t` flags, abbreviated to `-it`, run the container interactively, letting you type commands by hooking up to your terminal's _standard streams_.
- The `--name` flag gives your new container a descriptive name.
- The `--network` option connects your new `redis-client` container to the previously created virtual network, allowing it to communicate with the `redis-server` container. This way, both containers will receive hostnames corresponding to their names given by the `--name` option.
- By using the `-h` flag, you tell the Redis CLI to connect to a Redis server identified by its container name.

> **NOTE**
>
> There's a quicker way to connect your two containers through a virtual network without explicitly creating one. You can specify the `--link` option when running a new container:
>
> ```shell
> docker run --rm -it \
>            --name redis-client \
>            --link redis-server:redis-client \
>            redis redis-cli -h redis-server
> ```
>
> However, this option is deprecated and may be removed from Docker at some point.

When your new Docker container starts, you'll drop into an interactive Redis CLI which resembles a Python REPL with the following prompt:

```shell
redis-server:6379>
```

You can run simple commands to test the connection:

```shell
redis-server:6379> SET pi 3.14  # Set a key-value pair
OK
redis-server:6379> GET pi  # Retrieve the value
"3.14"
redis-server:6379> DEL pi  # Delete the key-value pair
(integer) 1
redis-server:6379> KEYS *  # List all keys
(empty array)
```

To exit the interactive Redis CLI, use the `exit` command or press `Ctrl + C`.

**Connecting to the Container Using Its IP Address**

If you installed Docker Desktop, then in most cases it won't route traffic from your host machine to the containers. There'll be no connection between your local network and the default Docker network ([Source][docker-traffic-routing]).

The same is true for Docker Desktop on Linux. On the other hand, if you're using Docker Engine or running Windows containers on a Windows host machine, then you'll be able to access such containers by their IP addresses.

Therefore, it may sometimes be possible for you to communicate with the Redis server directly from your host machine. First, find out the IP address of the corresponding Docker container:

```shell
docker inspect redis-server \
  -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{println}}{{end}}'
172.17.0.2
172.18.0.2
```

- If there is more than one IP address, this means that your container is connected to multiple networks. Containers get automatically connected to the default Docker network when you start them.

The addresses returned by the command may be different than the example above. You can use this IP address for the `-h` parameter's value instead of the linked container name in `redis-cli`. You can also use this IP address to connect to Redis with `netcat` or a Telnet client, like PuTTY or the `telnet` command:

```shell
telnet 172.17.0.2 6379
Trying 172.17.0.2...
Connected to 172.17.0.2.
Escape character is '^]'.
SET pi 3.14
+OK
GET pi
$4
3.14
DEL pi
:1
KEYS *
*0
^]
telnet> Connection closed.
```

- Remember to provide the port number, which defaults to `6379`, on which Redis listens for incoming connects.

You can type Redis commands in plaintext here because the server uses an unencrypted protocol unless you explicitly enable TLS support in the configuration.

Finally, you can take advantage of [port mapping][docker-port-mapping] to make Redis available outside of the Docker container. During development, you'll want to connect to Redis directly rather than through a virtual network from another container, so you don't have to connect it to any network just yet.

To use port mapping, stop and remove your existing `redis-server`, and then run a new container with the `-p` option:

```shell
docker stop redis-server
redis-server

docker rm redis-server
redis-server

docker run -d --name redis-server -p 6379:6379 redis
888d71ab723765c7fafa5ef79795bfde68fc5364dd739a535ab3c0dc3755299d
```

- The number on the left of the colon (`:`) represents the port number on the host machine or your computer, while the number on the right represents the mapped port inside the Docker container that's about to run.

Using the same port number on both sides effectively forwards it so that you can connect to Redis as if it were running locally on your computer:

```shell
telnet localhost 6379
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
INCR page_views
:1
INCR page_views
:2
INCR page_views
:3
^]
telnet> quit
Connection closed.
```

- To exit the connection, you must first escape (press `Ctrl + ]`) and then type `quit` or `Ctrl + C`.

After connecting to Redis, which is now visible on `localhost` and the default port, you can use the `INCR` command to increment the number of page views. if the underlying key doesn't exist, then Redis will initialize it with a value of 1.

> **NOTE**
>
> If you have Redis installed locally, or some system process is also using port `6379` on your host machine, then you'll need to map your port numbers differently using an unoccupied port. For example:
>
> ```shell
> docker run -d --name redis-server -p 9736:6379 redis
> ```
>
> - This will map port `9736` on your host machine to port `6379` inside the container. It doesn't matter which port you use, as long as it's available on your host machine.

Now that we can connect to Redis from the command line, we can see how to do it from a Python program.

### Connect to Redis from Python

At this point, you have a Redis server running in a Docker container, which can be accessed on localhost using the default port number for Redis. To learn more about the container, you can always retrieve valuable information by inspecting the object:

```shell
docker inspect redis-server
[
    {
        "Id": "888d71ab723765c7fafa5ef79795bfde68fc5364dd739a535ab3c0dc3755299d",
        ...
        "NetworkSettings": {
            ...
            "Ports": {
                "6379/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "6379"
                    }
                ]
            },
            ...
            "IPAddress": "172.17.0.2",
            ...
        }
    }
]
```

- The `docker inspect` command returns data in `JSON` format by default, which you can filter down further using [Go templates][docker-inspect-formatting].

Now, activate the project's virtual environment (inside the `page-tracker` directory) and start a new Python REPL:

```shell
$ source page-tracker/tracker-app-env/bin/activate

(tracker-app-env) $ python
```

Assuming the `redis` package was previously installed in the virtual environment, you should be able to import the Redis client for Python and all one of its method:

```shell
>>> from redis import Redis
>>> redis = Redis()
>>> redis.incr("page_views")
4
>>> redis.incr("page_views")
5
```

- When you create a new `Redis` instance without specifying any arguments, it'll try to connect to a Redis server running on `localhost` and the default port `6379`.
- In this case, calling `.incr()` confirms that you've successfully established a connection with Redis sitting in your Docker container because it remembered the last value of the `page_views` key.

> **NOTE**
>
> If the Redis client for Python hasn't been installed yet, with the virtual environment activated, install the `redis` package with `pip`:
>
> ```shell
> (tracker-app-env) $ pip install redis
> ```

If you need to connedct to Redis located on a remote machine, then supply a custom host and port number as parameters:

```shell
>>> from redis import Redis
>>> redis = Redis(host="127.0.0.1", port=6379)
>>> redis.incr("page_views")
6
```

Another way to connect to Redis is by using a specificallly formatted URL string:

```shell
>>> from redis import Redis
>>> redis = Redis.from_url("redis://localhost:6379/0")
>>> redis.incr("page_views")
7
```

- The `0` at the end of the URL string represents the database number, which is `0` by default. You can use different databases to separate different types of data. For newer versions of the `redis` package, the database number may be required.

With multiple ways to connect to a Redis instance, you can integrate Redis with your Flask application.

### Implement and Run the Flask App Locally

Open the `app.py` file in the `src/page_tracker` directory and add the following code:

```python
# src/page_tracker/app.py

from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis()

@app.get("/")
def index():
    page_views = redis.incr("page_views")
    return f"This page has been seen {page_views} times."
```

- After first importing the `flask` and `redis` packages as dependencies, the `Flask` application and `Redis` client are instantiated using default arguments. This means Redis will try to connect to a local server.
- The `index()` function is decorated with the `@app.get()` controller decorator, which is a shortcut for `@app.route("/", methods=["GET"])`. This function will handle `GET` requests whenever a user visits the root URL (`/`) of the web app.

The root endpoint increments the number of page views in Redis and displays a suitable message in the client's web browser. That's it! We have a complete starter web application that can handle HTTP traffic and persist state in a remote data store using fewer than ten lines of code.

To verify the Flask app is working as expected, issue the following command in the terminal:

```shell
(tracker-app-env) $ flask --app src.page_tracker.app run
 * Serving Flask app 'src.page_tracker.app'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
Press CTRL+C to quit
```

- The `flask --app` command (with the `src.page_tracker.app` reference) should be run within the `page-tracker` directory. Otherwise, you'll need to specify the full path to the `app.py` file.

- This should run the Flask development server on localhost and on port `5000` with debug mode disabled.

If you'd like to access the server from another computer on the same network, then you must bind it to all network interfaces by using the special address `0.0.0.0` instead of the default `localhost`, which represents the loopback interface:

```shell
(tracker-app-env) $ flask --app src.page_tracker.app run \
                    --host 0.0.0.0 \
                    --port 8080 \
                    --debug

 * Serving Flask app 'src.page_tracker.app'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:8080
 * Running on http://192.168.0.2:8080
Press CTRL+C to quit
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 759-189-650
```

You can also change the port number and enable debug mode with an appropriate command-line option or flag if you want to.

Once you've started the server, you can follow the link displayed in the terminal and see the page with the number of views in your web browser. Every time you refresh this page, the counter should increase by one.

Awesome! You've managed to create a bare-bones Flask app that tracks the number of page views using Redis. Next up, you'll learn how to test and secure your web application.

## Test and Secure Your Web Application

Before packaging and deploying any project to production, you should thoroughly test, examine, and secure the underlying source code. In this section, we'll exercise _unit, integration, and end-to-end_ tests. You'll also perform _static code analysis_ and _security scanning_ to identify potential issues and vulnerabilities when it's still cheap to fix them.

### Cover the Source Code with Unit Tests

_Unit testing_ involves testing a program's individual units or components to ensure that they work as expected. It's become a necessary part of the Software Development Lifecycle (SDLC) these days. Many engineers even take it a step further, rigorously following the [**test-driven-development**][test-driven-dev] methodology by writing their unit tests first to drive the code design.

When it comes to writing unit tests, it's quite common for those in the Python community (_Pythonistas_) to choose [`pytest`][pytest-module] over the standard library's `unittest` module. Thanks to the relative simplicity of `pytest`, this testing framework is quick to start with.

You can add `pytest` as an optional dependency to your project in the `pyproject.toml` file:

```toml
# pyproject.toml

[build-system]
requires = ["setuptools>=67.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "page-tracker"
version = "1.0.0"
dependencies = [
    "Flask",
    "redis",
]

[project.optional-dependencies]
dev = [
    "pytest",
]
```

You can group [optional dependencies][optional-dependencies] that are somehow related under a common name. For example, in the `.toml` file above, there's a group called `dev` to collect tools and libraries that you'll use during development. By keeping `pytest` separate from the main dependencies, you'll be able to install it on demand only when needed. After all, there's no point in bundling your tests or the associated testing framework with the built _distribution package_.

Don't forget to reinstall your Python package with the optional dependencies to get `pytest` into your project's virtual environment:

```shell
(tracker-app-env) $ pip install --editable ".[dev]"
```

- You can use square brackest to list the names of optional dependency groups defined in your `pyproject.toml` file. In this case, you ask to install the dependencies for development purposes, including a testing framework. Note that using quotes around the square brackets is recommended to prevent a potential filename expansion in the shell.

You don't have to keep the test modules in the same folder or the same namespace package as the code you're testing. You can create a separate directory branch for your tests, as follows:

```shell
page-tracker/
├── pyproject.toml
├── requirements.txt
├── src
│   └── page_tracker
│       ├── __init__.py
│       └── app.py
├── tracker-app-env/
└── test
    └── unit
        └── test_app.py
```

> **REMEMBER**
>
> You can create a new directory at the same time as creating a nested folder using `mkdir -p`:
>
> ```shell
> mkdir -p test/unit/
> ```

We place the `test_app.py` module inside the `test/unit/` folder to keep things organized. The `pytest` framework will discover your tests when you prefix them with the word `test`. Although this behavior can be changed, it's conventional to keep this format while mirroring each Python module with the corresponding test module. For example, to test the `app.py` module, you'll create a `test_app.py` module in the `test/unit/` folder.

Starting with the _happy path_ (when everything works as expected), we'll send a simple request to the server. Each Flask app comes with a convenient test client that you can use to make simulated HTTP requests. Because the test client doesn't require a live server to be running, your unit tests will execute much faster and will become more isolated.

You can get the test client and conveniently wrap it in a _test fixture_ (which initializes any preconditions a system may have which allows tests to be repeatable) to make it available to your test functions:

```python
# test/unit/test_app.py

import pytest

from page_tracker.app import app

@pytest.fixture
def http_client():
    return app.test_client()
```

- We first import `pytest` ot take advantage of its `@fixture` decorator against the custom `http_client` function. Choose this function's name carefully because it'll also become the name of the fixture that you can pass around as an argument to the individual test functions.

- We also import the Flask app from the `page_tracker` package to get the corresponding test client instance.

When you intend to write a unit test, you must always isolate it by eliminating any dependencies that your unit of code may have. This means that you should mock or stub out any external services, databases, or libraries that your code relies on. In this case, the Redis server is such a dependency.

Unfortunately, this app uses a hard-coded Redis client, which prevents mocking. This is a good argument for following test-driven development from the start, but it doesn't mean you have to go back and start over. Instead, we're going to refactor the code by implementing the dependency injection (where an object or function receives other objects or functions that it depends on) design pattern:

```python
# src/page_tracker/app.py

# The '-' in this file represent removed lines. The '+' represent added lines.

+from functools import cache

 from flask import Flask
 from redis import Redis

 app = Flask(__name__)
-redis = Redis()

 @app.get("/")
 def index():
-    page_views = redis.incr("page_views")
+    page_views = redis().incr("page_views")
     return f"This page has been seen {page_views} times."

+@cache
+def redis():
+    return Redis()
```

- Essentially, we move the Redis client creation code from the global scpe to a new `redis()` function, which your controller function calls at runtime on each incoming request. This allows your test case to substitute the returned Redis instance with a mock counterpart at the right time. But to ensure that there's only one instance of the client in memory, effectively making it a _singleton_ (i.e. the only instance in the program), you also cache the result of your new function.

Go back to your test module now and implement the following unit test:

```python
# test/unit/test_app.py

import unittest.mock

import pytest

from page_tracker.app import app

@pytest.fixture
def http_client():
    return app.test_client()

@unittest.mock.patch("page_tracker.app.redis")
def test_should_call_redis_incr(mock_redis, http_client):
    # Given
    mock_redis.return_value.incr.return_value = 5

    # When
    response = http_client.get("/")

    # Then
    assert response.status_code == 200
    assert response.text == "This page has been seen 5 times."
    mock_redis.return_value.incr.assert_called_once_with("page_views")
```

- We wrap the test function with Python's `@patch` decorator to inject a mocked Redis client into it as an argument.
- We also tell `pytest` to inject the HTTP test client fixture as another argument.
- The test function has a descriptive name that starts with the verb _should_ and follows the _Given-When-Then_ pattern. Both of these conventions, commonly used in _behavior-driven development_ (BDD), make your test read as **behavioral specifications**.

In your test case, you first set up the mock Redis client to always return 5 whenever its `.incr()` method gets called. Then, you make a forged HTTP request to teh root endpoint (`/`) and check the server's response status and body. Because mocking helps you test the _behavior_ of your unit, you only verify that the server calls teh correct method with the expected argument, trusting that the Redis client works correctly.

To execute your unit tests, you can either use the test runner integrated in your code editor, or you can type the following command:

```shell
(tracker-app-env) $ pytest -v test/unit/
```

- This command tells `pytest` to scan the `test/unit/` directory in order to look for test modules there. The `-v` flag increases the test reports _verbosity_ so that you can see more details about the individual test cases.

> **NOTE**
>
> The command above assumes that you're running it from the `page-tracker` directory. Otherwise, you'll need to specify the full path to the `test/unit/` directory.

Having your unit tests pass is only part of the story, however. Though they give you some level of confidence in your code, they're hardly enough to make any sort of guarantees.

After unit tests have passed, you should also run _integration tests_ to ensure that your code works well with other components. We'll add rudimentatary integration tests to our project next.

### Check Component Interactions Through Integration Tests

_Integration testing_ should be the next phase after running your unit tests. The goal of integration testing is to check how your components interact with each other as parts of a larger system.

We can reuse `pytest` to implement and run the integration tests. However, you'll install an additional `pytest-timeout` plugin to allow you to force the failure of test cases that taek too long to run:

```toml
# pyproject.toml

[build-system]
requires = ["setuptools>=67.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "page-tracker"
version = "1.0.0"
dependencies = [
    "Flask",
    "redis",
]

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-timeout",
]
```

Ideally, you don't need to worry about unit tests timing out because they should be optimized for speed. On the other hand, integration tests will take longer to run and could hang infinitely on a stalled network connection, preventing your test suite from finishing. This is why you should set a timeout for your integration tests.

Remember to reinstall your package with optional dependencies once again to make the `pytest-timeout` plugin available:

```shell
(tracker-app-env) $ pip install --editable ".[dev]"
```

Before continuing, add another subfolder for your integration tests and define a `conftest.py` file in your `test/` folder:

```shell
page-tracker/
│
├── src/
│   └── page_tracker/
│       ├── __init__.py
│       └── app.py
│
├── test/
│   ├── integration/
│   │   └── test_app_redis.py
│   │
│   ├── unit/
│   │   └── test_app.py
│   │
│   └── conftest.py
│
├── tracker-app-env/
│
├── requirements.txt
└── pyproject.toml
```

- You'll place common fixtures in `conftest.py` which different types of tests will share.

While your web app has just one component, you can think of Redis as another component that Flask needs to work with. Therefore, an integration test might look similar to your unit test, except that the Redis client won't be mocked anymore:

```python
# test/integration/test_app_redis.py

import pytest

@pytest.mark.timeout(1.5)
def test_should_update_redis(redis_client, http_client):
    # Given
    redis_client.set("page_views", 4)

    # When
    response = http_client.get("/")

    # Then
    assert response.status_code == 200
    assert response.text == "This page has been seen 5 times."
    assert redis_client.get("page_views") == b"5"
```

- Conceptually, your new test case consists of the same steps as before, but it interacts with the real Redis server. That's why you give the test at most 1.5 seconds to finish using the `@pytest.mark.timeout` decorator. The test function takes two fixtures as parameters:

  1. A Redis client connected to a local data store
  2. Flask's test client hooked to your web application

To make the second one available to your integration test as well, you must move the `http_client()` fixture from the `test_app` module to the `conftest.py` file:

```python
# test/conftest.py

import pytest
import redis

from page_tracker.app import app

@pytest.fixture
def http_client():
    return app.test_client()

@pytest.fixture(scope="module")
def redis_client():
    return redis.Redis()
```

- Because this file is located one level up in the folder hierarchy, `pytest` will pick up all the fixtures defined in it and make them visible throughout your nested folders.

- Apart from the `http_client()` fixture, which we moved from another Python module, we define a new fixture that returns a default Redis client. We set `scope="module"` for this fixture to reuse the same Redis client instance for all functions within a test module.

To perform your integration test, you'll have to double-check that a Redis server is running locally on the default port, `6379`. You can then start `pytest` as before, but point it to the folder with your integration tests:

```shell
(tracker-app-env) $ pytest -v test/integration/
```

Because the integration test connects to an actual Redis server, it'll overwrite the value that might have previously stored under the `page_views` key. However, if the Redis server isn't running while your integration tests are executing, or if Redis is running elsewhere, then your test will fail. This failure may be for the wrong reasons, making the outcome a _false negative_ error, as your code might actually be working as expected, it's just that the connection is bad.

To observe this problem, stop the Redis server and rerun the integration test:

```shell
(page-tracker) $ docker stop redis-server
redis-server
(page-tracker) $ pytest -v test/integration/

========================= short test summary info ==========================
FAILED test/integration/test_app_redis.py::test_should_update_redis -
⮑redis.exceptions.ConnectionError: Error 111 connecting to localhost:6379.
⮑Connection refused
============================ 1 failed in 0.19s =============================
```

- This uncovers an issue in your code, which doesn't gracefully handle Redis connection errors at the moment. In the spirit of test-driven development, you may first codify a test case that reproduces that problem and then fix it.

Add the following unit test in your `test_app` module with a mocked Redis client:

```python
# test/unit/test_app.py

import unittest.mock

from redis import ConnectionError

# ...

@unittest.mock.patch("page_tracker.app.redis")
def test_should_handle_redis_connection_error(mock_redis, http_client):
    # Given
    mock_redis.return_value.incr.side_effect = ConnectionError

    # When
    response = http_client.get("/")

    # Then
    assert response.status_code == 500
    assert response.text == "Sorry, something went wrong \N{pensive face}"
```

- You set the mocked `.incr()` method's side effect so that calling that method will raise the `redis.ConnectionError` exception, which you observed when the integration test failed.

Your new unit test, which is an example of a negative test, expects Flask to respond with an HTTP status code of `500` and a descriptive message.

Here's how to satisfy that unit test:

```python
# src/page_tracker/app.py

from functools import cache

from flask import Flask
from redis import Redis, RedisError

app = Flask(__name__)

@app.get("/")
def index():
    try:
        page_views = redis().incr("page_views")
    except RedisError:
        app.logger.exception("Redis error")
        return "Sorry, something went wrong \N{pensive face}", 500
    else:
        return f"This page has been seen {page_views} times."

@cache
def redis():
    return Redis()
```

- You intercept the top-level exception class, `redis.RedisError`, which is the ancestor of all exception types raised by the Redis client. If anything goes wrong, then you return the excepted HTTP status code and a message. For convenience, you also log the exception using the logger built into Flask.

> **NOTE**
>
> While a parent is the immediate base class that a child class is directly extending, an ancestor can be anywhere further up the inheritance hierarchy.

We've ammended our tests, implemented an integration test, and fixed a defect in our code after finding out about it, thanks to testing. Nonetheless, when you deploy your application to a remote environment, how will you know that all the pieces fit together and everything works as expected?

In the next section, you'll simulate a real-world scenario by performing an end-to-end test against your actual Flask server rather than the test client.

### Test a Real-World Scenario End to End (E2E)

End-to-end testing, also known as _broad stack testing_, encompasses many kinds of tests that can help you verify the system as a whole. This puts the complete software stack to the test by simulating an actual user's flow thorugh the application. Therefore, end-to-end testing requires a _deployment environment_ that mimics the production environment as closely as possible. A dedicated team of test engineers is usually needed, too.

> **NOTE**
>
> Because end-to-end tests have a high maintenance cost and tend to take a lot of time to set up and run, they sit atop Google's testing pyramid. In other words, you should aim for more integration tests, and even more unit tests in your projects.

As you'll eventually want to build a full-fledged _continuous integration pipeline_ for your Docker application, having some end-to-end tests in place will become essential. Start by adding another subfolder for your E2E tests:

```shell
page-tracker/
│
├── src/
│   └── page_tracker/
│       ├── __init__.py
│       └── app.py
│
├── test/
│   ├── e2e/
│   │   └── test_app_redis_http.py
│   │
│   ├── integration/
│   │   └── test_app_redis.py
│   │
│   ├── unit/
│   │   └── test_app.py
│   │
│   └── conftest.py
│
├── tracker-app-env/
│
├── requirements.txt
└── pyproject.toml
```

The test scenario you're about to implement will look similar to your integration test. The main difference though is that you'll be sending actual HTTP requests through the network to a live web server instead of relying on Flask's test client. To do so, you'll use the `requests` library, which you must first specify in your `pyproject.toml` file as another optional dependency:

```toml
# pyproject.toml

[build-system]
requires = ["setuptools>=67.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "page-tracker"
version = "1.0.0"
dependencies = [
    "Flask",
    "redis",
]

[project.optional-dependencies]
dev = [
    "pytest",
    "pytest-timeout",
    "requests",
]
```

- Since you won't be using `requests` to run your server in production, there's no need to require it as a regular dependency. Again, reinstall your Python package with optional dependencies using the editable mode:

```shell
(tracker-app-env) $ pip install --editable ".[dev]"
```

You can now use the installed `requests` library in your end-to-end test:

```python
# test/e2e/test_app_redis_http.py

import pytest
import requests

@pytest.mark.timeout(1.5)
def test_should_update_redis(redis_client, flask_url):
    # Given
    redis_client.set("page_views", 4)

    # When
    response = requests.get(flask_url)

    # Then
    assert response.status_code == 200
    assert response.text == "This page has been seen 5 times."
    assert redis_client.get("page_views") == b"5"
```

- This code is nearly identical to the integration test except for the following line:

  ```python
  response = requests.get(flask_url)
  ```

- We previously sent that request to the test client's root address, denoted with a slash character (`/`). Now, we don't know the exact domain or IP address of the Flask server, which may be running on a remote host. Therefore, the function now receives a Flask URL as an argument, which `pytest` injects as a fixture.

You may provide the specific web server's addresses through the command line. Similarly, your Redis server may be running on a different host, so you'll want to provide its address as a command-line argument as well. Currently though, your Flask app currently expects Redis to always run on the `localhost`. Let's update our code to make this more configurable:

```python
# src/page_tracker/app.py

import os
from functools import cache

from flask import Flask
from redis import Redis, RedisError

app = Flask(__name__)

@app.get("/")
def index():
    try:
        page_views = redis().incr("page_views")
    except RedisError:
        app.logger.exception("Redis error")
        return "Sorry, something went wrong \N{pensive face}", 500
    else:
        return f"This page has been seen {page_views} times."

@cache
def redis():
    return Redis.from_url(os.getenv("REDIS_URL", "redis://localhost:6379"))
```

- It's common to use environment variables for setting sensitive data, such as a database URL, because it provides an extra level of security and flexibility.
- In this more configurable version, the program expects a custom `REDIS_URL` variable to exist. If that variable isn't specified in the given environment, then you fall back to the default host and port.

To extend `pytest` with custom command-line arguments, you must edit `conftest.py` and hook into the framework's argument parser in the following way:

```python
# test/conftest.py

import pytest
import redis

from page_tracker.app import app

def pytest_addoption(parser):
    parser.addoption("--flask-url")
    parser.addoption("--redis-url")

@pytest.fixture(scope="session")
def flask_url(request):
    return request.config.getoption("--flask-url")

@pytest.fixture(scope="session")
def redis_url(request):
    return request.config.getoption("--redis-url")

@pytest.fixture
def http_client():
    return app.test_client()

@pytest.fixture(scope="module")
def redis_client(redis_url):
    if redis_url:
        return redis.Redis.from_url(redis_url)
    return redis.Redis()
```

- We define two optional arguments, `--flask-url` and `--redis-url`, using syntax similar to Python's [`argparse`][python-argparse] module.
- We then wrap these arguments in [session-scoped][pytest-fixture-scopes] fixtures, which you'll be able to inject into your test functions and other fixtures. Specifically, the existing `redis_client()` fixture now takes advantage of the optional Redis URL.

> **NOTE**
>
> Because your end-to-end integration tests rely on the same `redis_client()` fixture, you'll be able to connect to a remote Redis server by specifying the `--redis-url` option in both types of tests.

This is how you can run your end-to-end test with `pytest` by specifying the URL of the Flask web server and the corresponding Redis server:

```shell
(tracker-app-env) $ pytest -v test/e2e/ \
  --flask-url http://127.0.0.1:5000 \
  --redis-url redis://127.0.0.1:6379
```

- In this case, you can access both Flask and Redis through localhost (`127.0.0.1`) but your application could be deploeyd to a geographically distributed environment consisting of multiple remote machines.
- When you execute this command locally, make sure that Redis is running and you start your Flask server separately first:

```shell
(tracker-app-env) $ docker start redis-server
(tracker-app-env) $ flask --app page_tracker.app run
```

To improve code quality, you can keep adding more types of tests to your application if you have the capacity. Still, that usually takes a team of full-time quality assurance engineers. On the other hand, performing a code review or another type of _static code analysis_ is fairly low-hanging fruit that can uncover surprisingly many problems.

We'll look at this process now.

### Perform Static Code Analysis and Security Scanning

Now that the application works as expected, it's time to perform static code analysis without executing the underlying code. This helps to identify potential software defects and security risks in the code. While some steps of static analysis can be automated, others are usually done manually, for example through peer review.

We'll use the following automated tools added to the `pyproject.toml` file:

```toml
# pyproject.toml

[build-system]
requires = ["setuptools>=67.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "page-tracker"
version = "1.0.0"
dependencies = [
    "Flask",
    "redis",
]

[project.optional-dependencies]
dev = [
    "bandit",
    "black",
    "flake8",
    "isort",
    "pylint",
    "pytest",
    "pytest-timeout",
    "requests",
]
```

Don't forget to reinstall the dependencies:

```shell
(tracker-app-env) $ pip install --editable ".[dev]"
```
And pin them to the `requirements.txt` file:

```shell
(tracker-app-env) $ pip-chill --no-version > requirements.txt
```

> **NOTE**
>
> The original tutorial uses `pip freeze` (and most other projects will) to pin dependencies. `pip-chill --no-version` is used here since it streamlines dependency install in the future.

The added tools are utility tools which help with formatting and adhering to [PEP8][pep8] compliance:

```shell
(tracker-app-env) $ black src/ --check
would reformat /Users/roytelles/Desktop/Real Python/Docker/Dockerizing Flask/page-tracker/src/page_tracker/app.py

Oh no! 💥 💔 💥
1 file would be reformatted, 1 file would be left unchanged.

(tracker-app-env) $ isort src/ --check
ERROR: /home/.../app.py Imports are incorrectly sorted and/or formatted.

(tracker-app-env) $ flake8 src/
src/page_tracker/app.py:23:1: E302 expected 2 blank lines, found 1
```

- `black` flags any formatting inconsistencies in your code

- `isort` ensures that your `import` statements stay organized according to the [official recommendation][pep8-imports]

- `flake8` checks for any other PEP 8 style violations

If you don't see any output after running these tools, then there's nothing to fix! On the other hand, if warnings or errors appear, then you can correct any reported problems by hand or let those tools do it automatically when you omit the `--check` flag:

```shell
(tracker-app-env) $ black src/
reformatted /Users/roytelles/Desktop/Real Python/Docker/Dockerizing Flask/page-tracker/src/page_tracker/app.py

All done! ✨ 🍰 ✨
1 file reformatted, 1 file left unchanged.

(tracker-app-env) $ isort src/
Fixing /home/realpython/page-tracker/src/page_tracker/app.py

(tracker-app-env) $ flake8 src/
```

- Without the `--check` flag, both `black` and `isort` reformat the affected files in place without asking. Running these two commands also addresses PEP 8 compliance, causing `flake8` to no longer return any style violations.

> **NOTE**
>
> It's useful to keep the code tidy by following common code style conventions across your team. This way, when one person updates a source file, team members won't have to sort through changes to irrelevant parts of code, such as whitespace.

Once everything is clean, you can [_lint_][python-linters] your code to find potential _code smells_ &mdash; characteristics in the source code that indicate deeper problems &mdash; or ways to improve it:

```shell
(tracker-app-env) $ pylint src/
************* Module page_tracker.app
src/page_tracker/app.py:1:0: C0114: Missing module docstring (missing-module-docstring)
src/page_tracker/app.py:11:0: C0116: Missing function or method docstring (missing-function-docstring)
src/page_tracker/app.py:12:4: R1705: Unnecessary "else" after "return", remove the "else" and de-indent the code inside it (no-else-return)
src/page_tracker/app.py:22:0: C0116: Missing function or method docstring (missing-function-docstring)

-----------------------------------
Your code has been rated at 7.14/10
```

- When you run `pylint` against your web app's source code, it may start complaining about more or less useful things. It generally emits messages belonging to a few categories:

- E: Errors
- W: Warnings
- C: Convention violations
- R: Refactoring suggestions

Each remark has a unique identifier, such as `C0116`, which you can suppress if you don't find it helpful. You may include the suppressed identifiers in a global configuration file for a permanent effect or use a command-line switch to ignore certain errors on a given run. YOu can also add a specially formatted Python comment on a given line to account for special cases:

```python
# src/page_tracker/app.py

import os
from functools import cache

from flask import Flask
from redis import Redis, RedisError

app = Flask(__name__)

@app.get("/")
def index():
    try:
        page_views = redis().incr("page_views")
    except RedisError:
        app.logger.exception("Redis error")  # pylint: disable=E1101
        return "Sorry, something went wrong \N{pensive face}", 500
    else:
        return f"This page has been seen {page_views} times."

@cache
def redis():
    return Redis.from_url(os.getenv("REDIS_URL", "redis://localhost:6379"))
```

- In this case, we tell `pylint` to ignore a particular instance of the error `E1101` without suppressing it completely. It's a false positive because `.logger` is a dynamic attribute generated at runtime by Flask, which isn't available during a static analysis pass.

> **NOTE**
>
> If you intend to use `pylint` as part of your automated continuous integration pipeline, then you may want to specify when it should exit with an error code, which would typically stop the subsequent steps of the pipeline.
>
> For example, you can configure it to always return a neutral exit code:
>
> ```shell
> (tracker-app-env) $ pylint src/ --exit-zero
> ```
>
> This will never stop the pipeline from running, even when `pylint` finds some problems in the code. Alternatively, with `--fail-under`, you can specify an arbitrary score threshold at which `pylint` will exit with an error code.

You can see that `pylint` will give a score to your code and keeps track of it:

```shell
Your code has been rated at 7.14/10
```

When you fix a problem one way or another and run the tool again, then it'll report a new score and tell you how much it has improved or worsened. Use your best judgment to decide whether issues that `pylint` reports are worth fixing.

Finally, it's too common to inadvertently leak sensitive data through your source code or expose other security vulnerabilities. To reduce the risk of such incidents, you should perform security or vulnerability scanning of your source code before deploying it anywhere.

You can use `bandit` to scan your code for potential security issues:

```shell
(tracker-app-env) $ bandit -r src/
[main]  INFO    profile include tests: None
[main]  INFO    profile exclude tests: None
[main]  INFO    cli include tests: None
[main]  INFO    cli exclude tests: None
[main]  INFO    running on Python 3.11.4
Run started:2023-07-24 04:48:28.428300

Test results:
        No issues identified.

Code scanned:
        Total lines of code: 17
        Total lines skipped (#nosec): 0

Run metrics:
        Total issues (by severity):
                Undefined: 0
                Low: 0
                Medium: 0
                High: 0
        Total issues (by confidence):
                Undefined: 0
                Low: 0
                Medium: 0
                High: 0
Files skipped (0):
```

When you specify a path to a folder rather than to a file, you must include the `-r` flag to scan recursively. At this point, `bandit` shouldn't find any issues in your code. But, if you run it again after adding the following two lines at the bottom of your Flask application, then the tool will report issues with different severity levels:

```python
# src/page_tracker/app.py

# ...

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
```

- This [name-main idiom][name-main-idiom] is a common pattern found in many Flask applications because it makes development more convenient, letting you run the Python module directly. On the other hand, it exposes Flask's debugger, allowing the execution of arbitrary code, and binding to all network interfaces through the `0.0.0.0` address opens up your service to public traffic.

Therefore, to make sure your Flask app is secure, you should always run `bandit` or a similar tool before deploying the code to production.

Now, your web app is covered with unit, integration, and end-to-end tests. This means a number of automated tools have statically analyzed and modified its source code. Next, we'll continue on the path to continuous integration by wrapping the application in a Docker container so that you can deploy the whole project to a remote environment and faithfully replicate it on a local computer.

## Dockerize Your Flask Web Application

[dockerizing-flask-ci]: https://realpython.com/docker-continuous-integration/

[web-development]: https://realpython.com/learning-paths/become-python-web-developer/
[test-automation]: https://realpython.com/learning-paths/test-your-python-apps/
[python-redis]: https://realpython.com/python-redis/
[github]: https://realpython.com/python-git-github-intro/

[flask-app-resources]: https://realpython.com/bonus/docker-continuous-integration-code/

[docker-engine]: https://docs.docker.com/engine/
[docker-desktop]: https://docs.docker.com/desktop/

[pyproject-toml]: https://realpython.com/courses/packaging-with-pyproject-toml/
[setuptools]: https://setuptools.pypa.io/en/latest/
[build-backend]: https://peps.python.org/pep-0517/
[src-layout]: https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/

[repeatable-installs]: https://pip.pypa.io/en/stable/topics/repeatable-installs/

[volatile-memory]: https://en.wikipedia.org/wiki/Volatile_memory
[redis-persistence]: https://redis.io/docs/management/persistence/

[redis-image]: https://hub.docker.com/_/redis

[redis-cli]: https://redis.io/docs/ui/cli/
[docker-networks]: https://docs.docker.com/network/
[docker-bridge-network]: https://docs.docker.com/network/bridge/
[ephemeral-container]: https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#create-ephemeral-containers

[docker-traffic-routing]: https://docs.docker.com/docker-for-mac/networking/#i-cannot-ping-my-containers
[docker-port-mapping]: https://docs.docker.com/desktop/networking/#port-mapping

[docker-inspect-formatting]: https://docs.docker.com/config/formatting/

[test-driven-dev]: https://realpython.com/python-hash-table/#take-a-crash-course-in-test-driven-development
[pytest-module]: https://realpython.com/pytest-python-testing/
[optional-dependencies]: https://setuptools.pypa.io/en/latest/userguide/dependency_management.html#optional-dependencies

[python-argparse]: https://realpython.com/command-line-interfaces-python-argparse/
[pytest-fixture-scopes]: https://docs.pytest.org/en/6.2.x/fixture.html#fixture-scopes

[pep8]: https://realpython.com/python-pep8/
[pep8-imports]: https://peps.python.org/pep-0008/#imports
[python-linters]: https://realpython.com/python-code-quality/#linters
[name-main-idiom]: https://realpython.com/if-name-main-python/
