# Announcement

This repository is now archived as the project found a new home here: https://github.com/MISP/misp-docker
Note: for the time being docker images will still be pushed to https://hub.docker.com/repository/docker/ostefano/misp-docker so you just need to update the git remote to migrate to the new repository :)

# TAU's MISP Docker images

[![Build Status](https://img.shields.io/github/actions/workflow/status/ostefano/docker-misp/release-latest.yml)](https://hub.docker.com/repository/docker/ostefano/misp-docker)
[![Gitter chat](https://badges.gitter.im/gitterHQ/gitter.png)](https://gitter.im/MISP/Docker)

A production ready Dockered MISP based on CoolAcid's MISP Docker image (https://github.com/coolacid/docker-misp).

Like CoolAcid's MISP docker image, this is based on some of the work from the DSCO docker build, nearly all of the details have been rewritten.

-   Components are split out where possible, currently this is only the MISP modules
-   Over writable configuration files
-   Allows volumes for file store
-   Cron job runs updates, pushes, and pulls - Logs go to docker logs
-   Docker-Compose uses off the shelf images for Redis and MySQL
-   Images directly from docker hub, no build required
-   Slimmed down images by using build stages and slim parent image, removes unnecessary files from images

Additionally, this fork features the following improvements:

-   ARM (M1) support: move mariadb for increase compatibility
-   ARM (M1) support: move to updated and cross-platform mail exim4 image
-   Fix and improve support for cron jobs
-   Fix and improve support for syncservers
-   Fix supervisord process control (processes are correctly terminated upon reload)
-   Fix schema update by making it completely offline (no user interaction required)
-   Fix enforcement of permissions
-   Fix MISP modules loading of faup library
-   Fix MISP modules loading of gl library
-   Add support for new background job system (see https://github.com/MISP/MISP/blob/2.4/docs/background-jobs-migration-guide.md)
-   Add support for building specific MISP and MISP-modules commits
-   Add automatic configuration of sync servers (see `configure_misp.sh`)
-   Add autoamtic configuration of authentication keys (see `configure_misp.sh`)
-   Add direct push of docker images to Docker Hub
-   Consolidate docker compose files

As a result, this image is not for everybody and does not (and will not) fit every use case.
Nevertheless the underlying spirit of this fork is to allow "repeatable deployments", and all pull requests in this direction will be merged.

## Getting Started

-   Copy the `template.env` to `.env` 
-   Customize `.env` based on your needs (optional step)

### Run

-   `docker-compose pull` if you want to use pre-built images or `docker-compose build` if you want to build your own
-   `docker-compose up`
-   Login to `https://localhost`
    -   User: `admin@admin.test`
    -   Password: `admin`

### Configuration

The `docker-compose.yml` file allows further configuration settings:

```
"MYSQL_HOST=db"
"MYSQL_USER=misp"
"MYSQL_PASSWORD=example"    # NOTE: This should be AlphaNum with no Special Chars. Otherwise, edit config files after first run.
"MYSQL_DATABASE=misp"
"MISP_MODULES_FQDN=http://misp-modules" # Set the MISP Modules FQDN, used for Enrichment_services_url/Import_services_url/Export_services_url
"WORKERS=1"                 # Legacy variable controlling the number of parallel workers (use variables below instead)
"NUM_WORKERS_DEFAULT=5"     # To set the number of default workers
"NUM_WORKERS_PRIO=5"        # To set the number of prio workers
"NUM_WORKERS_EMAIL=5"       # To set the number of email workers
"NUM_WORKERS_UPDATE=1"      # To set the number of update workers
"NUM_WORKERS_CACHE=5"       # To set the number of cache workers
```

New options are added on a regular basis.

### Updating

Updating the images should be as simple as `docker-compose pull` which, unless changed in the `docker-compose.yml` file, will pull the latest built images.

### Production

-   It is recommended to specify which build you want to be running, and modify that version number when you would like to upgrade
-   Use docker-compose, or some other config management tool
-   Directory volume mount SSL Certs `./ssl`: `/etc/ssl/certs`
    -   Certificate File: `cert.pem`
    -   Certificate Key File: `key.pem`
    -   CA File for Cert Authentication (optional) `ca.pem`
-   Additional directory volume mounts:
    -   `./configs`: `/var/www/MISP/app/Config/`
    -   `./logs`: `/var/www/MISP/app/tmp/logs/`
    -   `./files`: `/var/www/MISP/app/files/`
    -   `./gnupg`: `/var/www/MISP/.gnupg/`
-   If you need to automatically run additional steps each time the container starts, create a new file `files/customize_misp.sh`, and replace the variable `${CUSTOM_PATH}` inside `docker-compose.yml` with its parent path.

## Versioning

GitHub builds the images automatically and pushes them to [Docker hub](https://hub.docker.com/r/ostefano/misp-docker). We do not use tags and versioning works as follows:

-   MISP (and modules) version specified inside the `template.env` file
-   Docker images are tagged based on the commit hash
-   Core and modules are tagged as core-commit-sha1[0:7] and modules-commit-sha1[0:7] respectively
-   The latest images have additional tags core-latest and modules-latest

## Image file sizes

-   Core server: 260MB
-   Modules: 470MB
