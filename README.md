Litestream/s6 Example
=====================

This repository provides an example of running a Go application in the same
container as Litestream by using [s6-overlay][]. This allows developers to
release their SQLite-based application and provide replication in a single
container.

[s6-overlay]: https://github.com/just-containers/s6-overlay


## How it works

The Go application provided in `main.go` will compile into a static binary
called `myapp` inside the Docker container. This application simply keeps
track of the number of page views in a SQLite database. A static build of
Litestream is downloaded into the container to provide replication to an
S3-compatible object store. Litestream is started and kept running via a small
init system called `s6`.

The Dockerfile is well-commented but this is the high level overview:

1. s6 is started via the entrypoint (`/init`)

1. s6 runs `etc/s6-overlay/scripts/litestream-restore` to restore the database
   if the database doesn't currently exist in `/data/db`.

1. s6 starts the `litestream` service via its `run` definition in the
   `etc/s6-overlay/s6-rc.d/litestream` directory. Litestream is automatically
   restarted on failure.

1. `myapp` is started inside the s6 environment through the use of Docker’s
   `CMD`. When `myapp` stops or crashes, s6 shuts down Litestream as well.


## Usage

### Prerequisites

Run the example container through the provided `docker-compose.yml`:

```sh
docker-compose up
```

This will start a copy of the container (after building) connected to a local
SFTP server provided by [`atmoz/sftp`](https://github.com/atmoz/sftp). This
connection is done by passing the `REPLICA_URL` environment variable to the
Docker container, following the [SFTP Replica Guide][].

[SFTP Replica Guide]: https://litestream.io/guides/sftp/


### Testing it out

In another window, you can run:

```sh
curl localhost:8080
```

and you should see:

```
This server has been visited 1 times.
```

Each time you run cURL, it will increment that value by one.


### Recovering your database

You can simulate a catastrophic disaster by stopping and removing your
container.

```
docker-compose down
```

When you restart the container again, it should print:

```
No database found, restoring from replica if exists
```

and then begin restoring from your replica. The visit counter on your app should
continue where it left off.

This can be contrasted with using `docker-compose stop` paired with
`docker-compose start`. Using `stop` and `start` will not remove the container
inbetween and the logs should print:

```
Database already exists, skipping restore
```
