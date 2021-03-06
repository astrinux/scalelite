[BigBlueButton](https://docs.bigbluebutton.org/) is an open source web conferencing system for online learning.  

# BigBlueButton

A single BigBlueButton server that meets the [minimum configuration](http://docs.bigbluebutton.org/2.2/install.html#minimum-server-requirements) supports around 150 concurrent users, which may be 3 meetings of 50 users, 5 meetings of 30 users, and so on.  Currently, the BigBlueButton project recommends that no single meeting exceed 100 users.


<p align="center">
  <img src="/images/simple.png"/>
</p><br>


Still, for many cases, the capacity of a single BigBlueButton server is sufficient.  However, what if a school wants to support 1,500 simultaneous users across 50 classes?  A single BigBlueButton server cannot handle load.

# Scalelite

Scalelite is an open source load balancer for BigBlueButton.  As a load balancer, Scalelite can manage a pool of BigBlueButton servers and, to the front-end application using BigBlueButton -- such as Moodle or Greenlight -- Scalelite can make that pool appear as a single, very scalable BigBlueButton server.

Scalelite constantly scans the servers in the pool and any new meetings are automatically started on the least loaded server in the pool.  More specifically, when Scalelite receives an incoming [create](http://docs.bigbluebutton.org/dev/api.html#create) API request, it automatically sends that request to the least loaded server in the pool.

For example, where a single BigBlueButton server can support 150 users, a pool 10 BigBlueButton servers can now support 1,500 users.

A pool of BigBlueButton servers can generate many recordings.  Scalelite's can consolidate these recordings together, index them, and, and respond very efficiently to incoming [getRecordings](https://docs.bigbluebutton.org/dev/api.html#getrecordings) API calls.  

Scalelite is a Ruby on Rails application.  To run Scalelite and its other components, we recommend you also use the [scalelite-run](https://github.com/blindsidenetworks/scalelite-run) project.  This project uses Docker and `docker-compose` to easily setup and run the components on a server.

<p align="center">
  <img src="/images/scalelite.png"/>
</p><br>

# Configuration

This section covers the common environment variables

## Environment Variables

### Required

* `URL_HOST`: The hostname that the application API endpoint is accessible from. Used to protect against DNS rebinding attacks.
* `SECRET_KEY_BASE`: A secret used internally by Rails. Should be unique per deployment. Generate with `rake secret`.
* `LOADBALANCER_SECRET`: The shared secret that applications will use when calling BigBlueButton APIs on the load balancer. Generate with `openssl rand -hex 32`
* `DATABASE_URL`: URL for connecting to the PostgreSQL database, see the [Rails documentation](https://guides.rubyonrails.org/configuring.html#configuring-a-database). Note that instead of using this environment variable, you can configure the database server in `config/database.yml`.
* `REDIS_URL`: URL for connecting to the Redis server, see the [Redis gem documentation](https://rubydoc.info/github/redis/redis-rb/master/Redis#initialize-instance_method). Note that instead of using this environment variable, you can configure the redis server in `config/redis_store.yml` (see below).

### Optional

* `PORT`: Set the TCP port number to listen on. Defaults to 3000.
* `BIND`: Instead of setting a port, you can set a URL to bind to. This allows using a Unix socket. See [The Puma documentation](https://puma.io/puma/Puma/DSL.html#bind-instance_method) for details.
* `INTERVAL`: Adjust the polling interval (in seconds) for updating server statistics and meeting status. Defaults to 60. Only used by the "poll" task.
* `WEB_CONCURRENCY`: The number of processes for the puma web server to fork. A reasonable value is 2 per CPU thread or 1 per 256MB ram, whichever is lower.
* `RAILS_MAX_THREADS`: The number of threads to run in the Rails process. The number of Redis connections in the pool defaults to match this value. The default is 5, a reasonable value for production.
* `RAILS_ENV`: Either `development`, `test`, or `production`. The Docker image defaults to `production`. Rails defaults to `development`.
* `BUILD_NUMBER`: An additional build version to report in the BigBlueButton top-level API endpoint. The Docker image has this preset to a value determined at image build time.
* `RAILS_LOG_TO_STDOUT`: Log to STDOUT instead of a file. Recommended for deployments with a service manager (e.g. systemd) or in Docker. The Docker image sets this by default.
* `REDIS_POOL`: Configure the Redis connection pool size. Defaults to `RAILS_MAX_THREADS`.

## Redis Connection (`config/redis_store.yml`)

For a deployment using docker, you should configure the Redis Connection using the `REDIS_URL` environment variable instead, see above.

The `config/redis_store.yml` allows specifying per-environment configuration for the Redis server.
The file is similar in structure to the `config/database.yml` file used by ActiveRecord.
By default, a minimal configuration is shipped which will connect to a Redis server on localhost in development, and use "fakeredis" (an in-memory Redis emulator) to run tests without requiring a Redis server.
The default production configuration allows specifying the Redis server connection to use via an environment variable, see below.
You may use this configuration file to set any of the options listed in the [Redis initializer](https://rubydoc.info/github/redis/redis-rb/master/Redis#initialize-instance_method).
Additionally, these options can be set:

* `pool`: The number of connections in the pool (should match number of threads). Defaults to `RAILS_MAX_THREADS` environment variable, otherwise 5.
* `pool_timeout`: Amount of time (seconds) to wait if all connections in the pool are in use. Defaults to 5.
* `namespace`: An optional prefix to apply to all keys stored in Redis.


# Administration

Scalelite comes with tools to add and remove BigBlueButton servers from the pool.

## Server Management

Server management is provided using rake tasks which update server information in Redis.

In a Docker deployment, these should be run from in the Docker container. You can enter the Docker container using a command like `docker exec -it <container name> /bin/sh`

### Show configured server details

```sh
./bin/rake servers
```

This will print a summary of details for each server which looks like this:

```
id: 2d2d674a-c6bb-48f3-8ad4-68f33a80a5b7
        url: https://bbb1.example.com/bigbluebutton/api
        secret: 2bdce5cbab581f3f20b199b970e53ae3c9d9df6392f79589bd58be020ed14535
        enabled
        load: 21.0
        online
```

Particular information to note:

* `id`: This is the ID value used when updating or removing the server
* `enabled` or `disabled`: Whether the server is administratively enabled. See "Enable/Disable servers" below.
* `load`: The number of meetings on the server. New meetings will be scheduled on servers with lower load. Updated by the poll process.
* `online`: Whether the server is responding to API requests. Updated by the poll process.

### Add a server

```sh
./bin/rake servers:add[url,secret]
```

The `url` value is the complete URL to the BigBlueButton API endpoint of the server. The `/api` on the end is required.
You can find the BigBlueButton server's URL and Secret by running `bbb-conf --secret` on the BigBlueButton server.

This command will print out the ID of the newly created server, and `OK` if it was successful.
Note that servers are added in the disabled state; see "Enable a server" below to enable it.

### Remove a server

```sh
./bin/rake servers:remove[id]
```

Warning: Do not remove a server which has running meetings! This will leave the database in an inconsistent state.
You should either wait for all meetings to end, or run the "Panic" function first.

### Disable a server

```sh
./bin/rake servers:disable[id]
```

Mark the server as disabled.
When a server is disabled, no new meetings will be started on the server.
Any existing meetings will continue to run until they finish.
The Poll process continues to run on disabled servers to update the "Online" status and detect ended meetings.
This is useful to "drain" a server for updates without disrupting any ongoing meetings.

### Enable a server

```sh
./bin/rake servers:enable[id]
```

Mark the server as enabled.

Note that the server won't be used for new meetings until after the next time the Poll process runs to update the load information.

### Panic a server

```sh
./bin/rake servers:panic[id]
```

Disable a server and clear all meeting state.
This method is used to recover from a crashed BigBlueButton server.
After the meeting state is cleared, anyone who tries to join a meeting that was previously on this server will instead be directed to a new meeting on a different server.

## Monitoring

### Check the status of the entire deployment
```sh
./bin/rake status
```

This will print a table displaying a list of all servers and some basic statistics that can be used for monitoring the overall status of the deployment

```
     HOSTNAME        STATE   STATUS  MEETINGS  USERS  LARGEST MEETING  VIDEOS 
 bbb1.example.com  enabled   online        12     25                7      15 
 bbb2.example.com  enabled   online         4     14                4       5 
```
