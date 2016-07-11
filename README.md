# Resalloc Homework

## Overview

This problem/solution consists of a few components:
(This is the only repo you will need to run them all, but links are provided to their source for reference)

**The mock problem space:**
  - 1 Docker container serving `fleet-acl` app (source: https://github.com/amp343/resalloc-fleet-acl) to act as the controller for server leasing
  - 5 Docker containers serving `fleet-server` app (source: https://github.com/amp343/resalloc-fleet-server) to simulate a fleet of leasable servers
**The important part: The CLI tool**
  - 1 local (host machine) cli tool `resalloc` (source: https://github.com/amp343/resalloc-cli)

{{ img }}

I wanted to use Docker here because the networking aspect of the problem is an
interesting detail that is aptly simulated with Docker for greater realism.

Ie, the `resalloc` tool should be able to affect changes across a real network.

## Environment & Assumptions

Note: these steps are specifically for an Ubuntu 14.04 machine, with assumptions:
  - 64-bit (check with `arch`)
    - **Reason:** CLI tool `resalloc` is compiled per-OS and arch; this specific build is for `linux`/`64`
  - Linux kernel > `3.10`(check with `uname -a`)
    - **Reason:** Docker will not run on kernel < `3.10`

## Getting Started

### Setup: Mock the problem space

These steps would not be required of a "real" end user of the `resalloc` tool,
(they would only need the `resalloc` binary) but are required here to mock the networking

- From a base Ubuntu 14.04 machine:
  - Install `docker-engine` (https://docs.docker.com/v1.8/installation/ubuntulinux/)
  - Install `docker-compose` (https://docs.docker.com/compose/install/)
  - Or, TLDR:
    - `sudo apt-get update`
    - `sudo apt-get install -y git curl wget libyaml-dev`
    - `wget -qO- https://get.docker.com/ | sh`
    - `sudo usermod -aG docker $(whoami)`
    - `sudo su -`
    - `curl -L https://github.com/docker/compose/releases/download/1.7.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose`
    - `chmod +x /usr/local/bin/docker-compose`
    - `git clone https://github.com/amp343/resalloc-hw.git && cd resalloc-hw`
    - `docker-compose up -d`

### Initial state

You should then see 6 containers running (confirm with `docker ps`):
  - `server{{1-5}}` -- the leasable server resources
  - `acl` -- the acl app that controls the leasing of servers

**Servers 1-3** are unleased and available for leasing immediately.
**Server 4** starts out leased to another user, but its lease is scheduled to expire
after 3 minutes, at which point it becomes available
**Server 5** starts out with a fresh 2-hour lease to another user, and will not be available for leasing during that time.

### The fun part -- start leasing!

Now you can use the `resalloc` CLI tool to lease resources

  - `./resalloc help` will display a help screen with available commands, but just to reiterate:
    - `./resalloc list` - list all server resources and their state
    - `./resalloc leased` - list any server resources leased by you
    - `./resalloc lease {{serverName}}` - lease a server from the pool
    - `./resalloc unlease {{serverName}}` - unlease a leased server

When using `./resalloc lease` or `./resalloc unlease`, the target server should be referred to by its `name`; ie, one of:
  - `server1`
  - `server2`
  - `server3`
  - `server4`
  - `server5`

For instance, `./resalloc lease server1`, or `./resalloc unlease server3`

### That's It - and once you have leased a server, confirm it's working

**Usage Example**

- `./resalloc lease server1` -- lease `server1`
- `./resalloc leased` -- see that the acl reports you have leased `server1`
- `docker ps` -- check the IP/port of the server you expect to have leased
- `curl {{ that }}` or request it in a browser -- see that the server itself indicates it is leased to you

**Note:** the user that has been provisioned for this exercise is `you@email.com`
If a server reports it is leased to this user, that means it is leased to you!

---

# App implementation details

### `resalloc` (https://github.com/amp343/resalloc-cli)

`resalloc` is a cli tool that makes requests to the `acl` app to perform
actions on the server resource fleet

I chose to write `resalloc` using Golang because of the directive "it should be
dead simple to run on a base Ubuntu 14.04 machine." Although I'm not that experienced
writing Golang (er... not at all), the fact that it cross-compiles natively for different
OS and ARCH makes it ideal for writing a cli tool that can be painlessly distributed.
So, it warrants usage for this situation despite lack of experience.

### `fleet-acl` (https://github.com/amp343/resalloc-fleet-acl)

`fleet-acl` is the app that is built to the `amp343/fleet-acl` Docker image and
runs on the `acl` container.

I chose to write `fleet-acl` using Ruby/Rails-API because Rails-API is well suited for quickly
bootstrapping a fairly robust application. The fact that `fleet-acl` should perform
authentication, manage a DB of stateful server resources records, and serve
those records restfully over an API to multiple clients (`resalloc` and `fleet-server`)
is what motivated that choice.

### `fleet-server` (https://github.com/amp343/resalloc-fleet-server)

`fleet-server` is a very minimal app that is build to the `amp343/fleet-server` Docker image
and lives on each of the available server containers (`server{{1-5}}`).

Basically, its purpose is to be leased to users as part of this exercise.

I chose to write `fleet-server` in PHP, well, just to mix it up. Upon each request,
a `fleet-server` consults the `fleet-acl` to learn about & report its status.

---

# Notes

## Focus of work

## Future/omitted work

Some things I would have liked to do with this problem, but couldn't due to time constraints:
- `resalloc` should refer to `acl`'s container name, instead of the known Docker ip/port
- `resalloc` should report more details on lease confict/failure, as there could be multiple reasons
- `resalloc` should be invoked as a usual binary and not by relative path `resalloc` vs `./resalloc`
- `resalloc` should have a niftier way of identifying the user's credentials, vs having them hardcoded relative.
- `resalloc` is probably missing out on some good Go patterns since I've not written Go before.
- `resalloc` has some duplicated code in actions; could be more modular
- `fleet-acl` could manage/report back to `resalloc` more details about accessing a leased server, vs having to look it up its ip manually with `docker ps`
- `fleet-acl` could have its responses cached
- `fleet-acl` should use a db in a separate container instead of a sqlite dev db in the same container
- `fleet-acl` should have any unused filesystem components cleared out to make it simpler
- `fleet-acl` could haver dryer, resource-based syntax in its router
- `fleet-acl` should expire resource leases on schedule, however, as implemented it only considers
  expiring leases upon receiving requests to the api. So, a stale lease could remain
  active beyond the 2 hour period if the `fleet-acl` api did not receive any additional requests

## Future tests

All the apps should have tests to prove that each them work as expected in isolation. For a time-constrained problem I'm erring on the side of just getting it to work. But in real life, it's definitely something that would be needed.

A quick, high-level test plan might look like:
- unit tests with high coverage goal for `resalloc`'s functions
- unit tests for `fleet-acl`'s controllers and models
- integration tests for `resalloc`'s api: happy path as well as anticipated failure paths
