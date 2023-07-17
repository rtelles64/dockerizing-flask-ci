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
