FROM python:3.11.2-slim-bullseye

# Get the latest updates and security patches
RUN apt-get update && \
    apt-get upgrade --yes

# This helps avoid the security risk of running as root
RUN useradd --create-home realpython
USER realpython
WORKDIR /home/realpython

ENV VIRTUALENV=/home/realpython/venv
RUN python3 -m venv $VIRTUALENV
ENV PATH="$VIRTUALENV/bin:$PATH"
