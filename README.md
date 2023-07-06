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

[dockerizing-flask-ci]: https://realpython.com/docker-continuous-integration/

[web-development]: https://realpython.com/learning-paths/become-python-web-developer/
[test-automation]: https://realpython.com/learning-paths/test-your-python-apps/
[python-redis]: https://realpython.com/python-redis/
[github]: https://realpython.com/python-git-github-intro/

[flask-app-resources]: https://realpython.com/bonus/docker-continuous-integration-code/

[docker-engine]: https://docs.docker.com/engine/
[docker-desktop]: https://docs.docker.com/desktop/