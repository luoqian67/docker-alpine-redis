### Alpine Linux Redis

A lightweight [Redis][redis] [Docker image][dockerhub_project] built from source atop [Alpine Linux][alpine_linux]. Available on [GitHub][github_project].

> If you're running Kubernetes 1.2.0 or later on all your cluster nodes, you should now use the non-`k8s` tags below. These tags are built on Alpine Linux 3.4, which adds the necessary DNS search support for service discovery. Kubernetes defaults to `dnsPolicy=ClusterFirst` in pod specs, and defines a single `nameserver` in `/etc/resolv.conf`. This means things should finally work correctly for Alpine Linux images without modification.

#### Stable 3.2.x Version Tags

- `3.2.4`, `3.2`, `3`, `stable`, `latest` (2016-09-26, [Dockerfile](https://github.com/sickp/docker-alpine-redis/tree/master/versions/3.2.3/Dockerfile), [Release notes][release_notes_3_2])
- `3.2.3` (2016-08-02)
- `3.2.2` (2016-07-28)
- `3.2.1` (2016-06-17)
- `3.2.0` (2016-05-06)

> NOTE: The default configuration in Redis 3.2.x binds to localhost and enables protected-mode. We don't want/need this since Docker networks provide isolation for containers. A simple workaround is to `bind 0.0.0.0` explicitly to disable this protection and revert to the previous behavior in 3.0.x. The images above set this in `/etc/redis.conf`.

#### Old 3.0.x Version Tags

- `3.0.7`, `3.0`, `old` (2016-01-28, [Dockerfile](https://github.com/sickp/docker-alpine-redis/tree/master/versions/3.0.7/Dockerfile), [Release notes][release_notes_3_0])
- `3.0.6` (2015-12-18)
- `3.0.5` (2015-10-15)

#### Kubernetes < 1.2.0

Tags with the `-k8s` suffix are built on [Alpine-Kubernetes 3.3][alpine_kubernetes], an image for Kubernetes and other Docker cluster environments that use DNS-based service discovery. It adds the necessary `search` domain support for DNS resolution.

- `3.2.0-k8s`, `3.2-k8s`, `3-k8s`, `stable-k8s`, `latest-k8s`
- `3.0.7-k8s`, `3.0-k8s`, `old-k8s`
- `3.0.6-k8s`

### Basic Usage

After the image name, just specify the executable to run followed by any options. By default, `redis-server /etc/redis.conf` will be executed.

> _NOTE_: If you want to add additional options to `redis-server`, ensure the first argument is the file path to your Redis configuration. Otherwise, Redis will generate a default configuration which is probably not what you want.

> _NOTE 2_: Do NOT override `ENTRYPOINT`. This image's entrypoint script fixes permissions on the data volume and becomes the `redis` user if you're running `redis-server`. Otherwise, it simply executes your command as is.

```bash
$ docker run --rm sickp/alpine-redis:3.2.4 # redis-server /etc/redis.conf
               _._                                                  
          _.-``__ ''-._                                             
     _.-``    `.  `_.  ''-._           Redis 3.2.4 (00000000/0) 64 bit
 .-`` .-```.  ```\/    _.,_ ''-._                                   
(    '      ,       .-`  | `,    )     Running in standalone mode
|`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
|    `-._   `._    /     _.-'    |     PID: 1
`-._    `-._  `-./  _.-'    _.-'                                   
|`-._`-._    `-.__.-'    _.-'_.-'|                                  
|    `-._`-._        _.-'_.-'    |           http://redis.io        
`-._    `-._`-.__.-'_.-'    _.-'                                   
|`-._`-._    `-.__.-'    _.-'_.-'|                                  
|    `-._`-._        _.-'_.-'    |                                  
`-._    `-._`-.__.-'_.-'    _.-'                                   
    `-._    `-.__.-'    _.-'                                       
        `-._        _.-'                                           
            `-.__.-'                                               

1:M 29 Sep 00:04:38.680 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 29 Sep 00:04:38.681 # Server started, Redis version 3.2.4
1:M 29 Sep 00:04:38.681 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 29 Sep 00:04:38.681 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 29 Sep 00:04:38.681 * The server is now ready to accept connections on port 6379
```

Explore the image in a container shell:

```bash
$ docker run --rm -it sickp/alpine-redis:3.2.4 ash
/data #
```

Check the server version:

```bash
$ docker run --rm sickp/alpine-redis:3.2.4 redis-server -v
Redis server v=3.2.4 sha=00000000:0 malloc=jemalloc-4.0.3 bits=64 build=2dbc115c06816d3e

$ docker run --rm sickp/alpine-redis:3.2.4 cat /etc/alpine-release
3.4.3
```

#### Example: Redis Server + CLI

This example starts a default Redis server in its own network. It requires Docker 1.9+ as it uses Docker's modern networking support. We then connect to it using the Redis command line interface.

##### Setup

Create a new network called `mynetwork` that will allow our containers to communicate in isolation.

```bash
$ docker network create mynetwork
bbeffd0ee68a977fb868eee76bc764a6f746ad11b8ec567c83a63ae71fc850d4

$ docker network ls
NETWORK ID          NAME                DRIVER
bd7fc5445682        host                host                
bbeffd0ee68a        mynetwork           bridge              
0527c4e41e56        bridge              bridge              
bad47935bd28        none                null  
```

##### Run Server

Next we create a simple Redis server instance, and give it the name `myserver`. Data is stored on the volume `/data`, which is automatically created for you on your development/container host. Because we're running in the foreground and will be removed with `--rm`, this volume will not dangle.

```bash
$ docker run --rm --net=mynetwork --name=myserver sickp/alpine-redis
...
1:M 09 May 18:55:40.710 * The server is now ready to accept connections on port 6379
```

##### Connect

In another terminal, we now connect to our server using the Redis CLI. This ephemeral container is connected to the `mynetwork` network and runs the command `redis-cli -h myserver`. Here we specify the hostname `myserver` which Docker has automagically inserted into this container's `/etc/hosts` file.

```bash
$ docker run --rm --net=mynetwork -it sickp/alpine-redis redis-cli -h myserver
myserver:6379> info
```

> NOTE: If the CLI fails to connect or gives you a warning about protected-mode, be sure to explicitly `bind 0.0.0.0` on the server.

#### Example: Redis Master / Slave + Data Containers

##### Setup

Create a new network called `mynetwork` that will allow our containers to communicate in isolation.

```bash
$ docker network create mynetwork
bbeffd0ee68a977fb868eee76bc764a6f746ad11b8ec567c83a63ae71fc850d4
```

Create data-only containers for the master and slave instances. This allows you to easily upgrade the server containers, while maintaining the persistent data stores.

```bash
$ docker create --name=redis-master-data sickp/alpine-redis
1996974d4891995d476e8f015972d10388cb301fe585217760e31edc8005aa5a

$ docker create --name=redis-slave-data sickp/alpine-redis
dd8f12ab673805468fc0dbd7dadedb37aa87c20732725a1530ff150963c41a6c
```

##### Run Master and Slave Servers

Start the master instance.

```bash
$ docker run --rm --name=redis-master --net=mynetwork --volumes-from=redis-master-data sickp/alpine-redis
```

Start the slave instance, setting `slaveof redis-master 6379`.

```bash
$ docker run --rm --name=redis-slave  --net=mynetwork --volumes-from=redis-slave-data  sickp/alpine-redis redis-server /etc/redis.conf --slaveof redis-master 6379
```

##### Connect

Connect to the master instance.

```bash
$ docker run --rm --net=mynetwork -it sickp/alpine-redis redis-cli -h redis-master
```

Connect to the slave instance.

```bash
$ docker run --rm --net=mynetwork -it sickp/alpine-redis redis-cli -h redis-slave
```


#### History

- 2016-09-29 - Updated to Redis 3.2.4 and Alpine Linux 3.4.3.
- 2016-08-09 - Updated to Redis 3.2.3 (and added Redis 3.2.2).
- 2016-06-30 - Updated to Redis 3.2.1.
- 2016-06-16 - Updated Redis 3.2.0 to Alpine Linux 3.4.0 (with `search` support for Kubernetes >=1.2.0).
- 2016-05-09 - Explicitly bind to all interfaces by default.
- 2016-05-06 - Added new stable Redis 3.2.0. Added more `-k8s` tags.
- 2016-02-09 - Added support for ALPINE_NO_RESOLVER in Kubernetes version.
- 2016-01-30 - Updated to Redis 3.0.7.
- 2016-01-27 - Added Kubernetes versions (-k8s), until Alpine Linux/musl adds DNS search support.
- 2015-12-29 - Official Docker Redis compatibility, and improved documentation.
- 2015-12-25 - Updated to Alpine Linux 3.3 (gcc 5.3.0), enable option passthrough to `redis-server`.
- 2015-12-18 - Updated to Redis 3.0.6.
- 2015-12-11 - Initial version.

[alpine_kubernetes]:  https://hub.docker.com/r/janeczku/alpine-kubernetes/
[alpine_linux]:       https://hub.docker.com/_/alpine/
[dockerhub_project]:  https://hub.docker.com/r/sickp/alpine-redis/
[github_project]:     https://github.com/sickp/docker-alpine-redis/
[redis]:              http://redis.io/
[release_notes_3_0]:  https://raw.githubusercontent.com/antirez/redis/3.0/00-RELEASENOTES
[release_notes_3_2]:  https://raw.githubusercontent.com/antirez/redis/3.2/00-RELEASENOTES
