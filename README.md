# Docker wait
`wait` is a wrapper script for Docker health checks that allows to use a shorter interval during the initialization
phase of a Docker container.

## Why?
Intervals for Docker health checks should normally not be set too low to avoid unnecessary load on the host.
However when starting a container that requires some initialization time, it takes at least the amount of time that
is specified via the `interval` parameter for the container to get a `healthy` status. This is unnecessary time lost
when waiting for the particular container to get healthy in order to perform subsequent tasks, especially when using
Docker Compose with service dependencies.

## How?
This script works around this issue by replacing Docker's `interval` parameter.
It is checking the container's health with an interval of 1 second during startup until the container gets healthy
for the first time. Afterwards it uses the interval in seconds specified as an argument.
In order for this to work, Docker's `interval` parameter has to be set to a low value like `1s`, otherwise the first
health check will not be triggered before the interval specified and an additional interval will be added between
the wrapper script executions. The "real" interval evaluation is implemented in the script itself by blocking until 
the interval time period is reached.
Because of the script blocking before executing the health check (after initialization phase),
Docker's `timeout` parameter has to be increased by the interval time period given to the script to achieve the desired
"real" timeout.

## Usage
```
wait INTERVAL COMMAND

Arguments:
  INTERVAL The interval in seconds between COMMAND executions after initialization phase
  COMMAND  The health check command to execute.
```

## Example
Imagine we have a health check script `check.sh` and we want to have a regular interval of `60s` with a timeout of
`30s`. The health check command would look as follows:

```
HEALTHCHECK --interval=1s --timeout=90s \
    CMD ["wait", "60", "check.sh"]
```

## Installation
The script can be directly downloaded into a Docker image via Dockerfile's `ADD` instruction:

```
ADD https://raw.githubusercontent.com/dzhgmbh/docker-wait/1.0/wait /usr/local/bin/
RUN chmod +x /usr/local/bin/wait
```

## Full example
Dockerfile:

```
FROM mysql:5.5

COPY healthcheck /usr/local/bin/
ADD https://raw.githubusercontent.com/dzhgmbh/docker-wait/1.0/wait /usr/local/bin/
RUN chmod +x /usr/local/bin/wait

HEALTHCHECK --interval=1s --timeout=90s --retries=3 \
    CMD ["wait", "60", "healthcheck"]
```
