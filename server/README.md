## QGIS server

### Warning

There are discussions on how to build these images and they are not considered stable. 
They are considered as NOT production ready.

### General information

The Docker image is built using *Ubuntu 20.04 (focal) and 22.04 (jammy)* and official QGIS DEBs from <https://qgis.org/>.
It includes *Nginx* and *Xvfb* and can be used as a standalone service (via HTTP TCP port 80) or as *FCGI* backend (via TCP port 9993).

### Requisites

You need **Docker >= 18.04** with `seccomp`. Support for the `statx` syscall, required by Qt 5.10+, may be made necessary in the future. This is generally included in **libseccomp >= 2.3.3**;
a kernel with `statx` support is also required; any kernel newer than 4.11 should be ok. Please check with your vendor.

Known good configurations are:
- Ubuntu 18.04.2+
- CentOS 8
- Fedora 29+

See <https://github.com/qgis/qgis-docker/issues/1> for further details.

Containers are not tested on hosts running OSes other than Linux.

### Services provided

This Docker container exposes HTTP on port `80` via Nginx and a direct FastCGI on port `9993` that can be used directly by an external HTTP proxy (like the provided `docker-compose.yml` does).
A sample Nginx configuration for using it as a *FastCGI* backend is also [provided](conf/nginx-fcgi-sample.conf).

### Available tags

Image name: `qgis/qgis-server`

- **QGIS stable**: `stable` | `stable-ubuntu`
- **QGIS LTR**: `ltr` | `ltr-ubuntu`

Example:

```sh
docker pull qgis/qgis-server:ltr
```

### Build the container

#### QGIS stable

```sh
# stable version
docker build -t qgis/qgis-server:stable .

# ltr version
docker build --build-arg repo=ubuntu-ltr -t qgis/qgis-server:ltr .
```

You may skip this step. The container will be downloaded from the Docker Hub.

### Run the docker and map host data

```sh
docker run -v $(pwd)/data:/io/data --name qgis-server -d -p 8010:80 qgis/qgis-server:ltr
```

`WMS` and `WFS` for a specific project will be published at `http://localhost:8010/ogc/<project_name>`.
An `/ows/` endpoint is also available for direct access to the `fcgi` (bypassing the `map=<project_name>` rewrite).
Direct access to `WFS3` is accessible via `http://localhost:8010/wfs3/`.


#### PostgreSQL connection service file (optional)

The [connection service file](https://www.postgresql.org/docs/12/libpq-pgservice.html) allows connection parameters to be associated with a single service name and thus to be able to use the same QGIS projects in different environments. This could also be achieved with [QGIS authentications](https://docs.qgis.org/3.16/en/docs/user_manual/auth_system/auth_workflows.html#database-authentication).
To use a pg_service file you need to bind mount it as shown in the [docker-compose](docker-compose.yml) or on run:

```sh
-v $(pwd)/conf/pg_service.conf:/etc/postgresql-common/pg_service.conf:ro
```


#### Plugins, fonts and SVG symbols (optional)

Plugins, custom fonts and SVG can be optionally exposed from host to the containers:

- plugins: `-v $(pwd)/plugins:/io/plugins`
- fonts: `-v $(pwd)/fonts:/usr/share/fonts`
- svg symbols: `-v $(pwd)/svg:/var/lib/qgis/.local/share/QGIS/QGIS3/profiles/default/svg`

Example:

```sh
docker run -v $(pwd)/data:/io/data -v $(pwd)/plugins:/io/plugins -v $(pwd)/fonts:/usr/share/fonts --name qgis-server -d -p 8010:80 qgis/qgis-server:ltr
```

#### Access the container via bash

```sh
docker exec -ti qgis-server /bin/bash
```

where `qgis-server` is the name of the container.

#### Logs and debugging

QGIS server log can retreived via `docker logs`

```sh
docker logs [-f] qgis-server
```

where `qgis-server` is the name of the container.

Default log level is set to `warning`. Log level can be increased during container deployment passing the `-e QGIS_SERVER_LOG_LEVEL=0` option:

```sh
docker run -e QGIS_SERVER_LOG_LEVEL=0 -v $(pwd)/data:/io/data -v $(pwd)/plugins:/io/plugins --name qgis-server -d -p 8010:80 qgis/qgis-server:ltr
```

### Run multiple instances of QGIS server

Adjust first the configuration in `conf/nginx.conf` with the proper number of expected workers
and `docker-compose.yml` with the path of data folder on the host.

Then:

```sh
docker-compose up -d --scale qgis-server=N
```

Where N is the number of expected QGIS server workers.


### Data dir structure

`$(pwd)/data` must have the following structure:

```text
data
 |
 |-- <project_name>
      |-- <project_name>.qgs
```

[oq-consolidate](https://github.com/gem/oq-consolidate) may helps you in exporting data suitable for QGIS server (consolidating project and layers).

`$(pwd)/plugins` must have the following structure:

```text
plugins
 |
 |-- <plugin_name>
      |-- <plugin_code>.py
      |-- metadata.txt
      |-- __init__.py
```

Custom fonts are loaded into `/usr/share/fonts`. `fc-cache` is run when container is started.

### Runtime customizations

The following variables can be customized during container deployment:

#### Nginx

When `SKIP_NGINX` is set to a different value than `0` or `false` the embedded copy of Nginx will not be started and an external reverse proxy is then required to access the FastCGI QGIS backend.

- `SKIP_NGINX`: default is *unset* (do not skip Nginx startup)

#### QGIS

- `QGIS_SERVER_LOG_LEVEL`: default is `1`
- `QGIS_SERVER_PARALLEL_RENDERING`: default is `true`
- `QGIS_SERVER_MAX_THREADS`: default is `2`
- `QGIS_SERVER_WMS_MAX_WIDTH`: not set by default
- `QGIS_SERVER_WMS_MAX_WIDTH`: not set by default

See [QGIS server documentation](https://docs.qgis.org/testing/en/docs/server_manual/config.html#environment-variables) for further details.

It is also possible to customized the ID of the user running QGIS server processes when it is required to write to host volumes (see [notes](#notes)):

- `QGIS_USER`: default is `nginx`, a numerical id must be proivided

Example: `docker run -e QGIS_USER=1000` or `docker run -e QGIS_USER=$(id -u qgis)`

### Notes

GeoPackages do not play well with multiple processes having gpkg files opened in `rw` mode. By default QGIS server processes lack write permission on `/io/data`.
If it is required to let QGIS server write data to `/io/data` make sure that either you are using a process-safe datastore (i.e. a Postgres backend) or you are
limiting horizontal scaling to one container only. QGIS server user can be customized via the `QGIS_USER` environment variable.
