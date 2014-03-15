# Node.js services with sysvinit and Monit

A simple example how to run a [node](http://nodejs.org) service on a server using a custom init.d script (to be able to easily start/stop the service as a daemon) and Monit (to monitor and restart the script if it is terminated).


## Deploying the node service

We assume that node is installed as `/usr/local/bin/node` and our node service behaves like this:

- The service should be called `node-service`
- The server script is `/home/node/node-service/server.js`
- The server listens on port 8000
- Output should be appended to `/var/log/node-service.log`


## Starting / stopping with init.d

A simple init.d script for the node-service is provided at `init.d/node-service`.

You may adjust your paths at the beginning:

    PATH=/sbin:/usr/sbin:/bin:/usr/bin:/usr/local/bin
    DAEMON_ARGS="/home/node/node-service/server.js" 
    DESC="node.js service"
    NODEUSER=node:node
    LOCAL_VAR_RUN=/var/run
    NAME=node
    DAEMON=/usr/local/bin/$NAME
    LOGFILE=/var/log/node-service.log

You can start the service with

    # /etc/init.d/node-service start

and terminate it with

    # /etc/init.d/node-service stop

Note: the server is not restarted if it crashes. This is a bit tricky with SysVinit, see the [node-init](https://github.com/nicokaiser/node-init) repository for a solution.


## Monitoring with Monit

### Installation

Monit can be used to monitor the service and restart it if it is terminated (e.g. crashes).

On Debian we just have to install the `monit` package and enable it in `/etc/default/monit`. On some systems, you also have to add this line to `/etc/monit/monitrc`:

    include /etc/monit/conf.d/*

### Monit basic configuration

Basic configuration for Monit is done in `/etc/monit/conf.d/basic`:

    set daemon 30
    set logfile /var/log/monit.log

    # set mailserver localhost
    # set mail-format { from: root@example.com }
    # set alert root@example.com

    set httpd port 2812 and
	    use address localhost
	    allow localhost

This way Monit checks its services every 30 seconds. To be notified about state changes, uncomment the mail* lines.


### Monit configuration for node-service

Monitoring for our service is configured in `/etc/monit/conf.d/node-service`:

    check node-service
    with pidfile /var/run/node-service.pid
    start program = "/etc/init.d/node-service start"
    stop program = "/etc/init.d/node-service stop"
    
    if failed port 8000 protocol HTTP request / with timeout 10 seconds then restart
    if 3 restarts within 5 cycles then timeout

This is quite self-explanatory. The service is restarted if it is not reachable via HTTP (or its pidfile is invalid), if it is restarted 3 times in 5*30 seconds, it is marked as timeout and not tried again.


## Running the service

Once Monit is running (after reboot or `/etc/init.d/monit start`), it starts checking if the node-service is running. If not, it starts the service automatically. 

The status can be displayed with

    # monit status

which outputs something like

    The Monit daemon 5.1.1 uptime: 48m
    
    Process 'node-service'
      status                            running
      monitoring status                 monitored
      pid                               6832
      parent pid                        1
      uptime                            53m 
      children                          0
      memory kilobytes                  9596
      memory kilobytes total            9596
      memory percent                    0.4%
      memory percent total              0.4%
      cpu percent                       0.0%
      cpu percent total                 0.0%
      port response time                0.001s to localhost:8000/ [HTTP via TCP]
      data collected                    Thu Mar 10 16:25:54 2011

To start/stop the node-service without Monit complaining about it, just use

    # monit start node-service
    # monit stop node-service


## References

The init.d script is based on the one by [Peter Horst](http://www.oghme.com/) on https://gist.github.com/715255

You can find a newer version for init.d at my [node-init](https://github.com/nicokaiser/node-init) repository.


## License

[MIT License](LICENSE.md)
