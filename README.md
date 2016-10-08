# twemproxies (nutcrackers)

**twemproxies** ( **nutcrackers** ) is a multithread, fast and lightweight proxy for [memcached](http://www.memcached.org/) and [redis](http://redis.io/) protocol. It was built primarily to reduce the number of connections to the caching servers on the backend. This, together with protocol pipelining and sharding enables you to horizontally scale your distributed caching architecture.

### QQ群：129951966

## Build

To build twemproxies from source with _debug logs enabled_ and _assertions enabled_:

    $ git clone git@github.com:twitter/twemproxies.git
    $ cd twemproxies
    $ autoreconf -fvi
    $ ./configure --enable-debug=full
    $ make
    $ src/nutcrackers -h

A quick checklist:

+ Use newer version of gcc (older version of gcc has problems)
+ Use CFLAGS="-O1" ./configure && make
+ Use CFLAGS="-O3 -fno-strict-aliasing" ./configure && make
+ `autoreconf -fvi && ./configure` needs `automake` and `libtool` to be installed

## Note

Try the follow command to get the twemproxies status:
	
    printf "status\r\n" | nc manage_ip manage_port
    
## Performance

We test twemproxis performance on our online physical machine.

##### Machine Configuration:

OS: Centos6 Linux 2.6.32-504.23.4.el6.x86_64

Cpu Model Name: Intel(R) Xeon(R) CPU E5-2630 v2 @ 2.60GHz

Cpu Physical: 2

Cpu Cores: 6

Cpu Processors: 24

Network: 10000Mbps

Memory: 128G

##### Test Example:

Command: set

Key: one million random keys (used redis-benchmark -r option)

Key length: 16

Value length: 3

##### Test Results:

| twemproxies thread number | redis-benchmark number | client connections | qps(k/s) | cpu(%) |
| -------------------------:| ----------------------:| ------------------:| --------:| ------:|
| 1                         | 1                      | 8                  |       45 |     90 |
| 2                         | 2                      | 16                 |       84 |    187 |
| 3                         | 3                      | 24                 |      121 |    285 |
| 4                         | 4                      | 32                 |      150 |    385 |
| 5                         | 5                      | 40                 |      186 |    478 |
| 6                         | 6                      | 48                 |      210 |    583 |
| 7                         | 7                      | 56                 |      235 |    681 |
| 8                         | 8                      | 64                 |      264 |    780 |
| 9                         | 9                      | 72                 |      289 |    878 |
| 10                        | 10                     | 80                 |      313 |    976 |
| 11                        | 11                     | 88                 |      336 |   1074 |
| 12                        | 12                     | 96                 |      354 |   1175 |
| 13                        | 13                     | 104                |      366 |   1280 |
| 14                        | 14                     | 112                |      386 |   1380 |
| 15                        | 15                     | 120                |      405 |   1480 |
| 16                        | 16                     | 128                |      422 |   1580 |
| 17                        | 17                     | 136                |      442 |   1681 |
| 18                        | 18                     | 144                |      460 |   1779 |
| 19                        | 19                     | 152                |      474 |   1880 |
| 20                        | 20                     | 160                |      490 |   1973 |

## Features

+ Multithread.
+ Fast.
+ Lightweight.
+ Maintains persistent server connections.
+ Keeps connection count on the backend caching servers low.
+ Enables pipelining of requests and responses.
+ Supports proxying to multiple servers.
+ Supports multiple server pools simultaneously.
+ Shard data automatically across multiple servers.
+ Implements the complete [memcached ascii](notes/memcache.md) and [redis](notes/redis.md) protocol.
+ Easy configuration of server pools through a YAML file.
+ Supports multiple hashing modes including consistent hashing and distribution.
+ Can be configured to disable nodes on failures.
+ Observability via stats exposed on the manage port.
+ Works with Linux, *BSD, OS X and SmartOS (Solaris)

## Help

    Usage: nutcrackers [-?hVdDt] [-v verbosity level] [-o output file]
                       [-c conf file] [-s manage port] [-a manage addr]
                       [-i interval] [-p pid file] [-m mbuf size]
					   [-T worker threads number]

    Options:
      -h, --help             : this help
      -V, --version          : show version and exit
      -t, --test-conf        : test configuration for syntax errors and exit
      -d, --daemonize        : run as a daemon
      -D, --describe-stats   : print stats description and exit
      -v, --verbose=N        : set logging level (default: 5, min: 0, max: 11)
      -o, --output=S         : set logging file (default: stderr)
      -c, --conf-file=S      : set configuration file (default: conf/nutcrackers.yml)
      -s, --port=N           : set manage port (default: 22222)
      -a, --addr=S           : set manage ip (default: 0.0.0.0)
      -i, --interval=N       : set interval in msec (default: 30000 msec)
      -p, --pid-file=S       : set pid file (default: off)
      -m, --mbuf-size=N      : set size of mbuf chunk in bytes (default: 16384 bytes)
	  -T, --thread_num=N     : set the worker threads number (default: 8)

## Zero Copy

In twemproxies, all the memory for incoming requests and outgoing responses is allocated in mbuf. Mbuf enables zero-copy because the same buffer on which a request was received from the client is used for forwarding it to the server. Similarly the same mbuf on which a response was received from the server is used for forwarding it to the client.

Furthermore, memory for mbufs is managed using a reuse pool. This means that once mbuf is allocated, it is not deallocated, but just put back into the reuse pool. By default each mbuf chunk is set to 16K bytes in size. There is a trade-off between the mbuf size and number of concurrent connections twemproxies can support. A large mbuf size reduces the number of read syscalls made by twemproxies when reading requests or responses. However, with a large mbuf size, every active connection would use up 16K bytes of buffer which might be an issue when twemproxies is handling large number of concurrent connections from clients. When twemproxies is meant to handle a large number of concurrent client connections, you should set chunk size to a small value like 512 bytes using the -m or --mbuf-size=N argument.

## Configuration

twemproxies can be configured through a YAML file specified by the -c or --conf-file command-line argument on process start. The configuration file is used to specify the server pools and the servers within each pool that twemproxies manages. The configuration files parses and understands the following keys:

+ **listen**: The listening address and port (name:port or ip:port) or an absolute path to sock file (e.g. /var/run/nutcrackers.sock) for this server pool.
+ **hash**: The name of the hash function. Possible values are:
 + one_at_a_time
 + md5
 + crc16
 + crc32 (crc32 implementation compatible with [libmemcached](http://libmemcached.org/))
 + crc32a (correct crc32 implementation as per the spec)
 + fnv1_64
 + fnv1a_64
 + fnv1_32
 + fnv1a_32
 + hsieh
 + murmur
 + jenkins
+ **hash_tag**: A two character string that specifies the part of the key used for hashing. Eg "{}" or "$$". [Hash tag](notes/recommendation.md#hash-tags) enable mapping different keys to the same server as long as the part of the key within the tag is the same.
+ **distribution**: The key distribution mode. Possible values are:
 + ketama
 + modula
 + random
+ **timeout**: The timeout value in msec that we wait for to establish a connection to the server or receive a response from a server. By default, we wait indefinitely.
+ **backlog**: The TCP backlog argument. Defaults to 512.
+ **tcpkeepalive**: A boolean value that controls if tcp keepalive enabled. Defaults to false.
+ **tcpkeepidle**: The time value in msec that a connection is in idle, and then twemproxy check this connection whether dead or not. 
+ **tcpkeepcnt**: The number of tcpkeepalive attempt check if one idle connection dead times when the client always had no reply. 
+ **tcpkeepintvl**: The time value in msec that the interval between every tcpkeepalive check when the client always had no reply.
+ **preconnect**: A boolean value that controls if twemproxies should preconnect to all the servers in this pool on process start. Defaults to false.
+ **redis**: A boolean value that controls if a server pool speaks redis or memcached protocol. Defaults to false.
+ **redis_auth**: Authenticate to the Redis server on connect.
+ **redis_db**: The DB number to use on the pool servers. Defaults to 0. Note: twemproxies will always present itself to clients as DB 0.
+ **server_connections**: The maximum number of connections that can be opened to each server. By default, we open at most 1 server connection.
+ **auto_eject_hosts**: A boolean value that controls if server should be ejected temporarily when it fails consecutively server_failure_limit times. See [liveness recommendations](notes/recommendation.md#liveness) for information. Defaults to false.
+ **server_retry_timeout**: The timeout value in msec to wait for before retrying on a temporarily ejected server, when auto_eject_host is set to true. Defaults to 30000 msec.
+ **server_failure_limit**: The number of consecutive failures on a server that would lead to it being temporarily ejected when auto_eject_host is set to true. Defaults to 2.
+ **servers**: A list of server address, port and weight (name:port:weight or ip:port:weight) for this server pool.


For example, the configuration file in [conf/nutcrackers.yml](conf/nutcrackers.yml), also shown below, configures 5 server pools with names - _alpha_, _beta_, _gamma_, _delta_ and omega. Clients that intend to send requests to one of the 10 servers in pool delta connect to port 22124 on 127.0.0.1. Clients that intend to send request to one of 2 servers in pool omega connect to unix path /tmp/gamma. Requests sent to pool alpha and omega have no timeout and might require timeout functionality to be implemented on the client side. On the other hand, requests sent to pool beta, gamma and delta timeout after 400 msec, 400 msec and 100 msec respectively when no response is received from the server. Of the 5 server pools, only pools alpha, gamma and delta are configured to use server ejection and hence are resilient to server failures. All the 5 server pools use ketama consistent hashing for key distribution with the key hasher for pools alpha, beta, gamma and delta set to fnv1a_64 while that for pool omega set to hsieh. Also only pool beta uses [nodes names](notes/recommendation.md#node-names-for-consistent-hashing) for consistent hashing, while pool alpha, gamma, delta and omega use 'host:port:weight' for consistent hashing. Finally, only pool alpha and beta can speak the redis protocol, while pool gamma, deta and omega speak memcached protocol.

    alpha:
      listen: 127.0.0.1:22121
      hash: fnv1a_64
      distribution: ketama
      auto_eject_hosts: true
      redis: true
      server_retry_timeout: 2000
      server_failure_limit: 1
      servers:
       - 127.0.0.1:6379:1

    beta:
      listen: 127.0.0.1:22122
      hash: fnv1a_64
      hash_tag: "{}"
      distribution: ketama
      auto_eject_hosts: false
      timeout: 400
      redis: true
      servers:
       - 127.0.0.1:6380:1 server1
       - 127.0.0.1:6381:1 server2
       - 127.0.0.1:6382:1 server3
       - 127.0.0.1:6383:1 server4

    gamma:
      listen: 127.0.0.1:22123
      hash: fnv1a_64
      distribution: ketama
      timeout: 400
      backlog: 1024
      preconnect: true
      auto_eject_hosts: true
      server_retry_timeout: 2000
      server_failure_limit: 3
      servers:
       - 127.0.0.1:11212:1
       - 127.0.0.1:11213:1

    delta:
      listen: 127.0.0.1:22124
      hash: fnv1a_64
      distribution: ketama
      timeout: 100
      auto_eject_hosts: true
      server_retry_timeout: 2000
      server_failure_limit: 1
      servers:
       - 127.0.0.1:11214:1
       - 127.0.0.1:11215:1
       - 127.0.0.1:11216:1
       - 127.0.0.1:11217:1
       - 127.0.0.1:11218:1
       - 127.0.0.1:11219:1
       - 127.0.0.1:11220:1
       - 127.0.0.1:11221:1
       - 127.0.0.1:11222:1
       - 127.0.0.1:11223:1

    omega:
      listen: /tmp/gamma
      hash: hsieh
      distribution: ketama
      auto_eject_hosts: false
      servers:
       - 127.0.0.1:11214:100000
       - 127.0.0.1:11215:1

Finally, to make writing a syntactically correct configuration file easier, twemproxies provides a command-line argument -t or --test-conf that can be used to test the YAML configuration file for any syntax error.

## Observability

Observability in twemproxies is through logs and stats.

twemproxies exposes stats at the granularity of server pool and servers per pool through the stats monitoring port. The stats are essentially JSON formatted key-value pairs, with the keys corresponding to counter names. By default stats are exposed on port 22222 and aggregated every 30 seconds. Both these values can be configured on program start using the -c or --conf-file and -i or --stats-interval command-line arguments respectively. You can print the description of all stats exported by  using the -D or --describe-stats command-line argument.

    $ nutcrackers --describe-stats

    pool stats:
      client_eof          "# eof on client connections"
      client_err          "# errors on client connections"
      client_connections  "# active client connections"
      server_ejects       "# times backend server was ejected"
      forward_error       "# times we encountered a forwarding error"
      fragments           "# fragments created from a multi-vector request"

    server stats:
      server_eof          "# eof on server connections"
      server_err          "# errors on server connections"
      server_timedout     "# timeouts on server connections"
      server_connections  "# active server connections"
      requests            "# requests"
      request_bytes       "total request bytes"
      responses           "# responses"
      response_bytes      "total response bytes"
      in_queue            "# requests in incoming queue"
      in_queue_bytes      "current request bytes in incoming queue"
      out_queue           "# requests in outgoing queue"
      out_queue_bytes     "current request bytes in outgoing queue"

Logging in twemproxies is only available when twemproxies is built with logging enabled. By default logs are written to stderr. twemproxies can also be configured to write logs to a specific file through the -o or --output command-line argument. On a running twemproxies, we can turn log levels up and down by sending it SIGTTIN and SIGTTOU signals respectively and reopen log files by sending it SIGHUP signal.

## Pipelining

twemproxies enables proxying multiple client connections onto one or few server connections. This architectural setup makes it ideal for pipelining requests and responses and hence saving on the round trip time.

For example, if twemproxies is proxying three client connections onto a single server and we get requests - 'get key\r\n', 'set key 0 0 3\r\nval\r\n' and 'delete key\r\n' on these three connections respectively, twemproxies would try to batch these requests and send them as a single message onto the server connection as 'get key\r\nset key 0 0 3\r\nval\r\ndelete key\r\n'.

Pipelining is the reason why twemproxies ends up doing better in terms of throughput even though it introduces an extra hop between the client and server.

## Deployment

If you are deploying twemproxies in production, you might consider reading through the [recommendation document](notes/recommendation.md) to understand the parameters you could tune in twemproxies to run it efficiently in the production environment.

## License

Copyright © 2016 VIPSHOP Inc.

Licensed under the Apache License, Version 2.0: http://www.apache.org/licenses/LICENSE-2.0
